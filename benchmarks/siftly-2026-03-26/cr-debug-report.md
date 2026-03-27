# Code Review & Debug Report — Siftly (Next.js + Prisma + SQLite)

**Reviewer**: Claude Code Review
**Date**: 2026-03-26
**Scope**: `lib/`, `app/api/`, `components/`, `middleware.ts`, `cli/`

---

## Executive Summary

47 findings across the codebase. 4 Critical, 8 High, 19 Medium, 16 Low. The most severe issues are: hardcoded Twitter API bearer tokens committed to source, plaintext API key storage in the database, an unauthenticated bookmarklet endpoint that bypasses all auth, and FTS5 injection via unsanitized search terms. Several race conditions exist in the categorization pipeline, and the Twitter API import route has no pagination limit allowing infinite loops.

---

## Critical Findings

### C-1: Hardcoded Twitter Bearer Tokens in Source Code
**File**: `lib/twitter-api.ts` line 5-6, `app/api/import/twitter/route.ts` line 4
**Description**: Two different Twitter/X internal bearer tokens are hardcoded directly in source files. These are credentials for Twitter's internal GraphQL API. If the repository is public (or becomes public), these tokens are immediately exposed.
**Code evidence**:
```typescript
// lib/twitter-api.ts:5
const BEARER = 'AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA'

// app/api/import/twitter/route.ts:4
const BEARER = 'AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I%2BxMb1nYFAA%3DUognEfK4ZPxYowpr4nMskopkC%2FDO'
```
**Impact**: Token exposure. Twitter could revoke these tokens, breaking the feature for all users. Additionally, two different bearer tokens exist across the codebase, suggesting one may be outdated or a different account entirely.
**Trace**: `lib/twitter-api.ts:fetchPage()` -> `lib/x-sync.ts:syncBookmarks()` -> `app/api/import/live/sync/route.ts` AND `app/api/import/twitter/route.ts:fetchPage()` -> its own POST handler. Two completely separate import code paths with duplicated logic and different tokens.

### C-2: API Keys Stored in Plaintext in SQLite Database
**File**: `app/api/settings/route.ts` lines 119-161, `app/api/categorize/route.ts` line 115-119
**Description**: Anthropic API keys, OpenAI API keys, Twitter auth tokens, and Twitter ct0 CSRF tokens are all stored as plaintext strings in the `Setting` table. The SQLite database file sits on disk unencrypted.
**Code evidence**:
```typescript
// app/api/settings/route.ts:125
await prisma.setting.upsert({
  where: { key: 'anthropicApiKey' },
  update: { value: trimmed },  // plaintext
  create: { key: 'anthropicApiKey', value: trimmed },
})

// app/api/import/live/route.ts:59-70 stores x_auth_token and x_ct0 in plaintext
```
**Impact**: If an attacker gains read access to the SQLite file (e.g., path traversal, backup exposure, Docker volume mount misconfiguration), all API keys and Twitter session cookies are immediately compromised. The Twitter auth_token + ct0 pair grants full access to a user's Twitter account.
**Trace**: `app/api/settings/route.ts:POST` stores keys -> `lib/settings.ts` reads them -> `lib/claude-cli-auth.ts:resolveAnthropicClient` uses `dbKey` parameter -> all AI operations. Also `app/api/import/live/route.ts:POST` stores Twitter cookies -> `lib/x-sync.ts:runScheduledSync` reads them for automated sync.

### C-3: Bookmarklet Endpoint Bypasses Authentication Entirely
**File**: `middleware.ts` lines 20-23, `app/api/import/bookmarklet/route.ts`
**Description**: The middleware explicitly exempts `/api/import/bookmarklet` from all authentication checks. This endpoint accepts arbitrary tweet data via POST from any origin (with CORS limited to x.com/twitter.com). However, the Origin header can be spoofed in server-to-server requests, and the endpoint performs no authentication of its own.
**Code evidence**:
```typescript
// middleware.ts:21-23
if (request.nextUrl.pathname === '/api/import/bookmarklet') {
  return NextResponse.next()
}
```
**Impact**: Anyone who knows the server URL can POST arbitrary bookmark data to the database without any authentication. This allows database pollution, storage exhaustion (no size limit on rawJson), and potential content injection.
**Trace**: External request -> `middleware.ts` (bypassed) -> `app/api/import/bookmarklet/route.ts:POST` -> `prisma.bookmark.create` with attacker-controlled data including `text`, `rawJson`, `tweetId`, etc.

### C-4: FTS5 Query Injection
**File**: `lib/fts.ts` lines 77-88
**Description**: The FTS5 search sanitization is insufficient. It strips `"`, `*`, `(`, `)` but does not strip FTS5 operators like `AND`, `OR`, `NOT`, `NEAR`, curly braces `{}`, column filters with `:`, or the `^` caret. An attacker can craft search terms that modify the query structure.
**Code evidence**:
```typescript
// lib/fts.ts:77-84
const terms = keywords
  .map((kw) => kw.replace(/["*()]/g, ' ').trim())
  .filter((kw) => kw.length >= 2)

// The terms are then joined raw:
const matchQuery = terms.join(' OR ')
```
A search for `text:secret` would restrict matching to the `text` column. `bookmark_id:somevalue` could leak bookmark IDs via the unindexed column. More dangerously, crafted FTS5 syntax could cause query errors or unexpected behavior.
**Impact**: Information disclosure through column-targeted searches, potential DoS through malformed FTS5 queries, or data leakage via the `UNINDEXED` bookmark_id column.

---

## High Findings

### H-1: Unbounded Twitter Import Loop — No Pagination Limit
**File**: `app/api/import/twitter/route.ts` lines 270-320
**Description**: The Twitter import POST handler fetches pages in a `while (true)` loop with no maximum page count. If the Twitter API keeps returning cursors (which it does for large bookmark sets), this request will run indefinitely.
**Code evidence**:
```typescript
// app/api/import/twitter/route.ts:270
while (true) {
  const data = await fetchPage(authToken.trim(), ct0.trim(), source, cursor, userId)
  // ...
  if (!nextCursor || tweets.length === 0) break
  cursor = nextCursor
}
```
Compare to `lib/x-sync.ts` which has `const MAX_PAGES = 50` as a safety limit.
**Impact**: Request timeout exhaustion, potential memory exhaustion (all imported data accumulates in the response), server hangs. For a user with thousands of likes, this could run for minutes or hours.
**Trace**: Client POST to `/api/import/twitter` -> `fetchPage` in infinite loop -> each page does N individual `prisma.bookmark.findUnique` + `prisma.bookmark.create` calls -> response never returns.

### H-2: Race Condition in Bookmark Category Update (Delete + Create Not Atomic)
**File**: `app/api/bookmarks/[id]/categories/route.ts` lines 21-32
**Description**: The category update operation first deletes all existing categories, then creates new ones. These are two separate operations, not wrapped in a transaction. If the server crashes or the request is interrupted between delete and create, the bookmark loses all categories permanently.
**Code evidence**:
```typescript
// Delete existing categories
await prisma.bookmarkCategory.deleteMany({ where: { bookmarkId: id } })

// Insert new categories (separate operation)
if (categoryIds.length > 0) {
  await prisma.bookmarkCategory.createMany({
    data: categoryIds.map((categoryId) => ({
      bookmarkId: id, categoryId, confidence: 1.0,
    })),
  })
}
```
**Impact**: Data loss on interrupted requests. Also, no validation that `categoryIds` actually exist — passing invalid category IDs will cause a foreign key constraint error after the delete has already succeeded.
**Trace**: `BookmarkCard.handleRemoveCategory` -> `PUT /api/bookmarks/{id}/categories` -> delete all -> potential crash -> categories gone. Also `BookmarkCard.handleSaveCategories` -> same path.

### H-3: No Bookmark ID Validation in Category Update
**File**: `app/api/bookmarks/[id]/categories/route.ts`
**Description**: The `PUT` handler takes a bookmark `id` from the URL path but never verifies the bookmark exists. It directly calls `deleteMany` and `createMany` with that ID. If the ID doesn't match a real bookmark, the delete is a no-op, but the create will insert orphan `BookmarkCategory` rows pointing to a non-existent bookmark.
**Impact**: Data integrity issue — orphan rows in `BookmarkCategory` table. While the schema has `onDelete: Cascade` from Bookmark to BookmarkCategory, there's no cascade from BookmarkCategory to Bookmark (it's the reverse direction).

### H-4: Race Condition in Categorization Pipeline Global State
**File**: `app/api/categorize/route.ts` lines 39-69, 97-99
**Description**: The categorization pipeline uses module-level global state (`globalState.categorizationState`) to track progress. The status check at lines 97-99 reads state twice without synchronization:
```typescript
if (getState().status === 'running' || getState().status === 'stopping') {
```
Since `getState()` creates a shallow copy each time, concurrent requests could see inconsistent state. More critically, the entire pipeline runs in a `void` async IIFE (line 152), meaning the POST handler returns immediately and the pipeline runs in the background — but if multiple POST requests arrive before the first one sets status to 'running', multiple pipelines can start simultaneously.
**Impact**: Multiple concurrent pipeline runs fighting over the same data, causing duplicate categorizations, wasted API calls, and potentially corrupt state.

### H-5: Timing-Based Auth Bypass in Middleware
**File**: `middleware.ts` lines 27-41
**Description**: The Basic Auth comparison uses plain string equality (`===`), which is vulnerable to timing side-channel attacks. An attacker can determine the correct password character-by-character by measuring response times.
**Code evidence**:
```typescript
if (user === username && pass === password) {
  return NextResponse.next()
}
```
**Impact**: In a network-accessible deployment, an attacker can progressively guess the password through timing analysis. For a local-only tool this is lower risk, but the middleware is designed specifically for network-exposed deployments (the comment mentions "Set SIFTLY_USERNAME and SIFTLY_PASSWORD in your .env to enable").

### H-6: Module-Level Settings Cache Not Process-Safe
**File**: `lib/settings.ts` lines 4-13
**Description**: The settings cache uses module-level variables (`_cachedModel`, `_cachedProvider`, etc.) with a 5-minute TTL. After a user changes settings via the UI, `invalidateSettingsCache()` is called. However, in serverless deployments (Vercel, AWS Lambda), each request may run in a different process/instance, so invalidation on one instance doesn't propagate to others.
**Impact**: After changing the AI provider or model, some requests may continue using the old provider/model for up to 5 minutes. In edge cases, requests might use the wrong AI provider entirely.

### H-7: SSRF Incomplete Protection in Link Preview
**File**: `app/api/link-preview/route.ts` lines 11-29
**Description**: The `isPrivateUrl` function blocks common private IP ranges, but it doesn't handle:
- IPv6 mapped IPv4 addresses (e.g., `::ffff:127.0.0.1`)
- `0.0.0.0/8` range beyond just `0.0.0.0` itself
- DNS rebinding attacks (where a public hostname resolves to a private IP)
- `[::1]` with brackets is checked, but `::1` without brackets is not handled by the regex
**Impact**: SSRF bypass allowing an attacker to scan internal network services through the link preview proxy.

### H-8: Export Loads All Bookmarks Into Memory
**File**: `lib/exporter.ts` lines 33-44, 146-147
**Description**: `fetchBookmarksFull()` calls `prisma.bookmark.findMany()` without any pagination or streaming. For a user with 50,000+ bookmarks, this loads all data (including `rawJson` which can be large) into memory at once.
**Code evidence**:
```typescript
async function fetchBookmarksFull(where?: object): Promise<BookmarkRow[]> {
  return prisma.bookmark.findMany({
    where,
    include: { mediaItems: true, categories: { include: { category: true } } },
    orderBy: { importedAt: 'desc' },
  }) as Promise<BookmarkRow[]>
}
```
**Impact**: Out-of-memory crashes on large datasets. The ZIP export is even worse because it additionally downloads all media files sequentially (line 133: `const fileData = await downloadFile(item.url)`).

---

## Medium Findings

### M-1: Duplicate Code Between twitter-api.ts and import/twitter/route.ts
**File**: `lib/twitter-api.ts` and `app/api/import/twitter/route.ts`
**Description**: These two files contain nearly identical implementations of `fetchPage`, `parsePage`, `extractMedia`, `tweetFullText`, `bestVideoUrl`, and `decodeHtmlEntities`. They even have different bearer tokens. Any fix applied to one must be manually duplicated to the other.
**Impact**: Maintenance burden, inconsistent behavior, divergent bug fixes.

### M-2: Duplicate Code in bookmarklet/route.ts
**File**: `app/api/import/bookmarklet/route.ts`
**Description**: The bookmarklet route re-implements `tweetFullText`, `extractMedia`, `bestVideoUrl` locally rather than importing from `lib/twitter-api.ts`. Missing `decodeHtmlEntities` call means HTML entities in note tweets remain escaped.
**Impact**: Bookmarks imported via bookmarklet will have different text processing (e.g., `&amp;` instead of `&` in note tweets).

### M-3: AI Search Cache Grows Unbounded Within TTL Window
**File**: `app/api/search/ai/route.ts` lines 21-23
**Description**: The cache eviction only triggers when size reaches 100 entries, and it evicts the oldest entry. But within the 5-minute TTL, entries are never cleaned up proactively. With rapid diverse queries, the cache Map could accumulate substantial data in memory (each entry stores full bookmark hydration data).
**Impact**: Memory leak proportional to unique queries within each TTL window.

### M-4: FTS Table Rebuild Loads All Bookmarks Into Memory
**File**: `lib/fts.ts` lines 34-42
**Description**: `rebuildFts()` calls `prisma.bookmark.findMany()` without pagination to load ALL bookmarks before inserting into the FTS table. This should use cursor-based pagination like other batch operations in the codebase.
**Impact**: Memory exhaustion on large datasets during FTS rebuild.

### M-5: No Input Size Limit on Import Endpoints
**File**: `app/api/import/route.ts`, `app/api/import/bookmarklet/route.ts`
**Description**: The file import endpoint reads the entire uploaded file into memory (`jsonString = await file.text()`) with no size check. The bookmarklet endpoint reads the entire JSON body without limit. An attacker could POST a multi-GB payload.
**Impact**: Memory exhaustion, denial of service.

### M-6: Sequential Database Operations in Import (N+1 Query Pattern)
**File**: `app/api/import/route.ts` lines 77-117, `app/api/import/twitter/route.ts` lines 274-316
**Description**: Both import handlers check for existing bookmarks one-by-one inside a loop (`prisma.bookmark.findUnique` per tweet), then create them one-by-one. For 10,000 bookmarks, this is 10,000 find queries + up to 10,000 create operations.
**Impact**: Extremely slow imports, database contention, long-running requests.

### M-7: `backfillEntities` Lacks Cursor-Based Pagination
**File**: `lib/rawjson-extractor.ts` lines 263-287
**Description**: The `backfillEntities` function queries `where: { entities: null }` without cursor-based pagination. If `entities: null` applies to many rows, the same rows may be re-fetched after they're updated (since the `null` filter now excludes them). However, there's a subtle issue: it also updates rows one-by-one in a loop, not in a batch. With 10,000 bookmarks this is 10,000 individual UPDATE queries.
**Impact**: Performance — O(n) individual queries instead of batch operations.

### M-8: Category Slug Collision Not Fully Prevented
**File**: `app/api/categories/route.ts` lines 84-93
**Description**: The slug generation (`generateSlug`) can produce the same slug for different names (e.g., "AI Tools" and "AI  Tools" both become "ai-tools"). The check does `findFirst` with `OR: [{name}, {slug}]`, but there's a TOCTOU race: two simultaneous requests could both find no existing category and then both try to create, with the second failing on the unique constraint.
**Impact**: Uncaught unique constraint error returns a 500 instead of a user-friendly 409.

### M-9: `enriched` Counter Race Condition in enrichAllBookmarks
**File**: `lib/vision-analyzer.ts` lines 461, 541-542
**Description**: The `enriched` counter is incremented from multiple concurrent batch tasks (line 549: `runWithConcurrency(batchTasks, ENRICH_CONCURRENCY)`) without synchronization:
```typescript
enriched++
onProgress?.(enriched)
```
While JS is single-threaded, the `await` points in each task allow interleaving, and the `enriched++` / `onProgress` pair is not atomic — another task could increment between these two lines.
**Impact**: Progress reporting may show incorrect/duplicate counts. Not a data corruption issue since JS is single-threaded, but the reported count may skip values.

### M-10: OpenAI Assistant Role Silently Drops Image Content
**File**: `lib/ai-client.ts` lines 86-88
**Description**: When an assistant message contains image blocks, the OpenAI client silently filters them out, keeping only text blocks:
```typescript
if (m.role === 'assistant') return {
  role: 'assistant' as const,
  content: parts.map(p => p.type === 'text' ? p : p)
    .filter((p): p is OpenAI.ChatCompletionContentPartText => p.type === 'text')
}
```
The `map(p => p.type === 'text' ? p : p)` is a no-op (it returns `p` regardless), then the filter drops all non-text parts.
**Impact**: If the system ever sends assistant messages with images (e.g., in multi-turn conversations), the images are silently dropped. Currently no code path does this, but it's a latent bug.

### M-11: Vision Analyzer Retries Don't Back Off on Non-Rate-Limit Errors
**File**: `lib/vision-analyzer.ts` lines 139-166
**Description**: The retry logic treats all "retryable" errors the same (500, 502, 503, ECONNREFUSED, ETIMEDOUT, etc.) but the backoff delays are fixed at [1500, 4000, 10000]ms. Server overload errors (529) should have longer backoff, while network errors should retry faster.
**Impact**: Suboptimal retry behavior — either too aggressive for overloaded servers or too slow for transient network glitches.

### M-12: `exportCategoryAsZip` Downloads Media Sequentially
**File**: `lib/exporter.ts` lines 128-137
**Description**: Media files are downloaded one-by-one in a for loop with `await downloadFile(item.url)`. Each has a 15-second timeout. For a category with 100 media items, this could take up to 25 minutes.
**Impact**: Request timeout, poor user experience.

### M-13: DELETE /api/bookmarks Deletes Categories Along with Bookmarks
**File**: `app/api/bookmarks/route.ts` lines 14-30
**Description**: The "clear all bookmarks" handler also deletes all categories (`prisma.category.deleteMany({})`). This may be intentional, but it means user-created custom categories are lost when clearing bookmark data.
**Impact**: Unexpected data loss — users might expect "clear bookmarks" to preserve their category structure.

### M-14: Inconsistent `tweetCreatedAt` Date Handling on Invalid Input
**File**: `app/api/import/bookmarklet/route.ts` line 154-155, `app/api/import/twitter/route.ts` line 296-297
**Description**: Both routes create a `new Date(tweet.legacy.created_at)` without checking if the result is valid (`isNaN`). If `created_at` is a non-empty but unparseable string, an "Invalid Date" object is passed to Prisma.
**Impact**: Potential database errors or silently stored invalid dates. The `lib/parser.ts` version correctly handles this (line 88-89), but the API routes don't.

### M-15: Pipeline Categorize Queue Drain Logic Can Skip Items
**File**: `app/api/categorize/route.ts` lines 219-252
**Description**: The `drainCategorizeQueue` function has complex locking via `catFlushing` boolean. The non-final path returns early if `catPending.length < CAT_BATCH_SIZE`. If the total number of bookmarks is not a multiple of `CAT_BATCH_SIZE` (25), the last partial batch is only processed in the `final=true` drain. However, if a worker throws between the last `catPending.push` and the `drainCategorizeQueue()` call, items could be lost.
**Impact**: Some bookmarks may not get categorized in a single pipeline run.

### M-16: `runWithConcurrency` Has No Error Isolation
**File**: `lib/vision-analyzer.ts` lines 221-238
**Description**: If one task throws in `runWithConcurrency`, the worker stops and the error propagates to `Promise.all`. This causes all other workers to be abandoned (their remaining tasks are never executed).
**Impact**: A single failed vision analysis can abort the entire batch, leaving many items unprocessed.

### M-17: Search Index Leaks Internal Bookmark IDs to AI Prompt
**File**: `app/api/search/ai/route.ts` lines 121, 326
**Description**: The AI search prompt includes internal database IDs (`b.id`) which are CUIDs. The AI then returns these IDs in its response. While not directly exploitable, it exposes internal identifiers in AI API calls, and the AI could hallucinate IDs that match other bookmarks.
**Impact**: Information disclosure (internal IDs sent to third-party AI APIs), potential for ID confusion if AI hallucinates.

### M-18: `hasAnthropicKey` Check Is Wrong
**File**: `app/api/settings/route.ts` line 40
**Description**: The settings GET handler returns `hasAnthropicKey: anthropic !== null`. But `anthropic` is the result of `findUnique` which returns `null` if the key doesn't exist AND returns the Setting object if it does — even if the value is empty. A setting with key `anthropicApiKey` but empty value would report `hasAnthropicKey: true`.
**Impact**: UI might show "key configured" when the key is actually empty.

### M-19: Codex CLI Output File Not Cleaned Up on Timeout
**File**: `lib/codex-cli.ts` lines 46-75
**Description**: When the `execFileAsync` call times out (line 49: `timeout: timeoutMs`), the error catch block tries to read and clean up the temp file. But if the process is killed mid-write, the temp file may exist as a partially-written file in `/tmp` and is only cleaned up in the catch block's try/catch. If the `readFileSync` succeeds but returns partial data, it's returned as a "successful" result.
**Impact**: Partial/corrupt AI responses treated as valid, temp file accumulation in `/tmp`.

---

## Low Findings

### L-1: Theme Toggle FOUC (Flash of Unstyled Content)
**File**: `components/theme-toggle.tsx`
**Description**: The theme state is initialized to `false` (dark) and then updated in `useEffect` after mount. On a light-theme page, the toggle briefly shows the wrong icon before the effect runs.

### L-2: `cachedCategories` Module-Level State in Client Component
**File**: `components/bookmark-card.tsx` lines 193-216
**Description**: `cachedCategories` is a module-level variable in a client component. In development mode with React StrictMode, effects run twice, which could cause a double fetch. In production, the cache persists for the lifetime of the browser tab and is never invalidated.
**Impact**: Stale category data if categories are created/deleted in another tab.

### L-3: `previewCache` Never Cleaned Up
**File**: `components/bookmark-card.tsx` line 34
**Description**: The `previewCache` Map grows indefinitely as the user scrolls through bookmarks. Each bookmark card that has a link triggers a cache entry. No eviction strategy.
**Impact**: Memory usage grows linearly with pages viewed.

### L-4: Command Palette Thumbnail Uses Raw Twitter URL
**File**: `components/command-palette.tsx` line 146, 161
**Description**: The command palette uses `b.mediaItems[0]?.thumbnailUrl` directly as an `<img src>` without proxying through `/api/media`. This will fail due to Twitter's CORS/hotlink restrictions in many scenarios.
**Impact**: Broken thumbnail images in the command palette.

### L-5: `maskKey` Can Show Negative Stars for Short Keys
**File**: `app/api/settings/route.ts` lines 5-9
**Description**: For keys between 8 and 10 characters, `'*'.repeat(raw.length - 10)` produces `'*'.repeat(negative)` which returns an empty string. The function works but reveals 10 characters of the key (6 prefix + 4 suffix) for keys that are exactly 10 characters long, which may be too much.

### L-6: Dead `like` Query ID Placeholder
**File**: `app/api/import/twitter/route.ts` line 44
**Description**: `queryId: 'REPLACE_ME'` — the Likes import feature ships with a placeholder query ID and will always fail with a 400 error from Twitter.

### L-7: `extractKeywords` Strips Non-ASCII Characters
**File**: `lib/search-utils.ts` line 13
**Description**: `.replace(/[^a-z0-9\s]/g, ' ')` strips all non-ASCII characters, making it impossible to search for bookmarks in CJK, Arabic, Cyrillic, or any non-Latin script.

### L-8: `analyzeItem` Video Type Prefix Injection Produces Invalid JSON
**File**: `lib/vision-analyzer.ts` lines 202-208
**Description**: For video thumbnails, the code replaces the opening `{` with `{"_type":"video_thumbnail",`. If the AI response already starts with whitespace before the `{`, the regex `^\{` won't match, and the prefix is not injected. If the response has nested `{` objects, only the outermost is replaced. Both edge cases could produce invalid JSON.

### L-9: `isPrivateUrl` Doesn't Handle Uppercase Hostnames
**File**: `app/api/link-preview/route.ts` line 14
**Description**: `hostname` from `new URL()` is already lowercased per URL spec, so this is not actually a bug. However, the `fd` prefix check for IPv6 ULA (`/^fd[0-9a-f]{2,}:/i`) uses a regex that requires a colon after the hex digits, but IPv6 addresses in URL hostnames are in brackets `[fd00::1]`. The regex will never match a real IPv6 ULA in a URL hostname.

### L-10: `formatCsvField` Doesn't Handle Newlines
**File**: `lib/exporter.ts` lines 46-49
**Description**: The CSV field formatter wraps in double quotes and escapes `"`, but doesn't escape newlines. Tweet text often contains newlines, which will break CSV row parsing in many readers.

### L-11: FTS Table Hardcoded in Multiple Places
**File**: `lib/fts.ts` lines 11, 57
**Description**: The FTS table name `bookmark_fts` is defined as a constant on line 11 but the INSERT statement on line 57 uses the literal string `bookmark_fts` instead of the constant. If the table name is ever changed, the insert would break.

### L-12: `detectTools` Substring Match Is Too Greedy
**File**: `lib/rawjson-extractor.ts` lines 147-151
**Description**: The subdomain match uses `domain.endsWith(knownDomain)`. This means `evilgithub.com` would match `github.com`, and `notvercel.com` would match `vercel.com`.

### L-13: `mindmap` Node Position Calculation Uses Tweet Index Not ID
**File**: `app/api/mindmap/route.ts` lines 145-183
**Description**: Tweet node positions are calculated purely from array index, not from any stable property. Reloading the mindmap after a bookmark is added/removed will shuffle all node positions.

### L-14: No Rate Limiting on Any API Endpoint
**Description**: None of the API routes implement rate limiting. The AI search endpoint triggers an LLM call per request. The categorize endpoint can be repeatedly triggered. The import endpoints can be called in rapid succession.
**Impact**: API cost amplification, denial of service.

### L-15: `cheerio`-less HTML Parsing with Regex in Link Preview
**File**: `app/api/link-preview/route.ts` lines 51-57, 211-238
**Description**: OG meta tags are extracted using regex against raw HTML. This is fragile and can be defeated by tag ordering, attributes, or encoding variations. For example, `content` attributes containing `>` or `'` will break the regex.

### L-16: Prisma Schema Missing Index on `mediaItems.imageTags`
**File**: `prisma/schema.prisma`
**Description**: Multiple queries filter on `imageTags: null` (vision analyzer) or `imageTags: { contains: kw }` (search), but there's no index on the `imageTags` column. With many media items, these queries will do full table scans.

---

## Systematic Impact Tracing for Critical/High Findings

### C-1 Impact Chain (Hardcoded Bearer Tokens)
```
Source: lib/twitter-api.ts:5, app/api/import/twitter/route.ts:4
  -> Used in: lib/twitter-api.ts:fetchPage() line 101
    -> Called by: lib/x-sync.ts:syncBookmarks() line 20
      -> Called by: app/api/import/live/sync/route.ts POST
      -> Called by: lib/x-sync.ts:runScheduledSync() line 114
        -> Called by: setInterval in startScheduler() line 87
  -> Used in: app/api/import/twitter/route.ts:fetchPage() line 118
    -> Called by: app/api/import/twitter/route.ts POST line 271
```
If tokens are revoked: All live sync (manual + scheduled) and direct Twitter import break simultaneously. Users with running schedulers will see errors logged every interval.

### C-3 Impact Chain (Unauthenticated Bookmarklet)
```
External POST to /api/import/bookmarklet
  -> middleware.ts bypasses auth (line 21-23)
  -> route.ts processes any JSON body with tweet data
    -> prisma.bookmark.create (stores rawJson of arbitrary size)
    -> prisma.mediaItem.createMany (stores arbitrary URLs)
  -> Downstream: these fake bookmarks enter:
    -> Categorization pipeline (wastes AI API calls)
    -> FTS search index (pollutes search results)
    -> Export (included in CSV/JSON/ZIP exports)
    -> Mindmap visualization
    -> Stats counts
```
The blast radius is the entire application. Every feature that reads bookmarks is affected by injected data.

### H-1 Impact Chain (Unbounded Import Loop)
```
POST /api/import/twitter with source='like' and valid credentials
  -> while(true) loop in route handler
    -> Each iteration: HTTP call to Twitter + N DB queries
    -> A user with 50,000 likes: ~500 pages x 100 tweets
    -> Each tweet: 1 findUnique + 1 create + potentially 1 createMany for media
    -> ~75,000+ DB operations, ~500 HTTP calls
    -> Next.js request timeout (default 60s on Vercel) will kill it mid-write
    -> Partial import: some bookmarks imported, rest lost
    -> No import job tracking (unlike /api/import/route.ts)
    -> User cannot resume — re-running will skip already-imported tweets but
       has no cursor state to resume from where it left off
```

### H-4 Impact Chain (Pipeline Race Condition)
```
Two rapid POST requests to /api/categorize:
  Request 1: getState().status === 'idle' -> true -> setState({status:'running'})
  Request 2: getState().status === 'idle' -> true (before Request 1's setState propagates)
  -> Both start void async pipelines
  -> Both call seedDefaultCategories() (potential unique constraint conflict)
  -> Both fetch same unprocessed bookmarks
  -> Both call AI APIs for same bookmarks (double cost)
  -> Both call writeCategoryResults for same bookmarks (upsert so no data corruption, but wasted work)
  -> Both set status to 'idle' when done, but the second one's completion handler
     may overwrite the first one's error/progress state
```

---

## Summary Table

| Severity | Count | Key Themes |
|----------|-------|------------|
| Critical | 4 | Hardcoded credentials, plaintext key storage, auth bypass, query injection |
| High | 8 | Unbounded loops, race conditions, timing attacks, SSRF, memory exhaustion |
| Medium | 19 | Code duplication, N+1 queries, cache issues, error handling, data integrity |
| Low | 16 | UI glitches, missing indexes, regex fragility, missing i18n support |

---

## Top 5 Recommendations (Priority Order)

1. **Move all secrets to environment variables or encrypted storage.** Remove hardcoded bearer tokens, encrypt API keys at rest in the database, never store raw Twitter session cookies.

2. **Add authentication to the bookmarklet endpoint.** Use a per-installation secret token in the bookmarklet URL, or implement a signed request scheme.

3. **Add pagination limits to all unbounded loops.** Especially `app/api/import/twitter/route.ts` (add MAX_PAGES), `lib/fts.ts:rebuildFts` (add cursor pagination), `lib/exporter.ts` (add streaming/pagination).

4. **Fix FTS5 injection.** Properly escape all FTS5 special characters and operators, or use parameterized FTS5 queries.

5. **Wrap the category update in a transaction and validate inputs.** Verify bookmark exists, verify all categoryIds exist, use `prisma.$transaction` for the delete+create pair.
