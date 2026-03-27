# AFM Fusion v3.2 Report: Siftly

**Target**: siftly-v32 — Next.js + Prisma + SQLite Twitter/X bookmark manager with AI categorization
**Scope**: lib/, app/api/, components/, middleware.ts, cli/

---

## Phase 1: Seed Map

| # | File:Line | Seed |
|---|-----------|------|
| S1 | middleware.ts:21 | Bookmarklet endpoint bypasses all auth — middleware returns NextResponse.next() unconditionally for /api/import/bookmarklet |
| S2 | middleware.ts:29-34 | Basic Auth credential comparison uses `===` (not constant-time) — timing side-channel |
| S3 | app/api/import/bookmarklet/route.ts:110-177 | Bookmarklet POST has CORS allowing x.com/twitter.com origins but no authentication — any script running on x.com can write to the database |
| S4 | app/api/import/twitter/route.ts:270 | `while (true)` loop fetching Twitter API pages with no max-page limit — unbounded pagination |
| S5 | lib/twitter-api.ts:5-6 | Hardcoded Twitter Bearer token in source code |
| S6 | app/api/import/twitter/route.ts:4 | Second different hardcoded Bearer token — diverges from lib/twitter-api.ts |
| S7 | lib/fts.ts:14-23 | `$executeRawUnsafe` with string interpolation for FTS_TABLE constant |
| S8 | lib/fts.ts:78-84 | FTS5 MATCH query sanitizes `"*()` but not all FTS5 operators (e.g., `^`, `NEAR`, `NOT`, column filters with `:`) |
| S9 | app/api/settings/route.ts (all POST) | API keys stored in plaintext in SQLite Settings table |
| S10 | app/api/categorize/route.ts:112-119 | API key from request body is persisted to database — fire-and-forget credential injection |
| S11 | lib/settings.ts:3-8 | Module-level cache with 5-minute TTL — stale provider/model after settings change |
| S12 | lib/x-sync.ts:72 | `syncing` flag is module-level boolean — no protection against concurrent Node.js processes |
| S13 | app/api/import/twitter/route.ts:245-260 | Twitter authToken and ct0 accepted directly from request body over HTTP — credentials in transit |
| S14 | app/api/import/live/route.ts:58-71 | X auth credentials (auth_token, ct0) stored in plaintext in Settings table |
| S15 | lib/vision-analyzer.ts:71 | CONCURRENCY = 12 for vision API calls — potential rate-limit exhaustion |
| S16 | app/api/bookmarks/[id]/categories/route.ts:21 | No validation that bookmark `id` actually exists before deleting its categories |
| S17 | lib/categorizer.ts:212 | Regex `\[[\s\S]*\]` for JSON extraction is greedy — will match from first `[` to last `]` in the entire AI response |
| S18 | app/api/import/bookmarklet/route.ts:4-8 | CORS origin whitelist is good but `corsHeaders` defaults to `https://x.com` when origin isn't in the set, rather than rejecting |
| S19 | app/api/link-preview/route.ts:11-29 | SSRF protection: `isPrivateUrl` checks IP ranges but not DNS rebinding or IPv6-mapped IPv4 |
| S20 | No file | No rate limiting on any API endpoint — all routes are unthrottled |
| S21 | app/api/categorize/route.ts:152-370 | Fire-and-forget async pipeline via `void (async () => { ... })()` — unhandled rejection risk |
| S22 | lib/fts.ts:30-62 | `rebuildFts` loads ALL bookmarks into memory at once before batching inserts |
| S23 | app/api/import/twitter/route.ts:35-55 | ENDPOINTS.like.queryId = 'REPLACE_ME' — placeholder left in production code |
| S24 | lib/rawjson-extractor.ts:147-149 | `domain.endsWith(knownDomain)` substring match for tool detection — false positives (e.g., `notgithub.com` matches `github.com`) |
| S25 | app/api/media/route.ts:3-9 | Media proxy allowlist is static — only twimg.com domains; any new media host would bypass and get 403 |
| S26 | middleware.ts:49-54 | Matcher regex excludes _next/ and static files but does not exclude /api routes from Basic Auth — correct, but means ALL API calls need auth headers |

---

## Phase 2: Assume + Verify (The Board)

### S1: Bookmarklet endpoint bypasses auth — VULNERABLE

**Premise**: "The bookmarklet needs to work cross-origin from x.com, so it can't send Basic Auth credentials."

**Mechanism**: middleware.ts:21 — exact pathname match `/api/import/bookmarklet` causes `NextResponse.next()`, skipping all auth. The bookmarklet endpoint at app/api/import/bookmarklet/route.ts accepts POST with a JSON body containing raw `TweetResult[]` objects and writes them directly to the database.

**Effect**: When Basic Auth is enabled (SIFTLY_USERNAME + SIFTLY_PASSWORD set), the bookmarklet endpoint is the ONLY unauthenticated write path into the system. Any entity that can reach the server on this endpoint can inject arbitrary bookmark data into the database.

**WHY (multi-level)**:
- **L1**: Developers assumed the bookmarklet would only be called from their own browser extension/bookmarklet on x.com. CORS origin checking exists (x.com, twitter.com only).
- **L2**: But CORS is a browser-enforced mechanism. Any non-browser client (curl, script, bot) ignores CORS entirely. The endpoint has zero server-side authentication.
- **L3**: Meta-pattern: "CORS-as-auth" — relying on browser origin enforcement as a substitute for server-side authentication.

**ABSTRACT search**: Grep for other endpoints that rely on CORS alone for protection.

**Impact**: If the instance is exposed to the internet (the reason auth exists), anyone can POST crafted tweet data to `/api/import/bookmarklet`, inserting arbitrary text into the bookmark database. This could inject malicious URLs (stored in `rawJson`, displayed in UI), pollute categorization, or fill storage.

**Counter-evidence check**: Does the CORS origin check provide meaningful protection? No — CORS only restricts browser JavaScript. Server-to-server requests, `curl`, or any HTTP client bypasses CORS.

---

### S2: Basic Auth timing side-channel — HOLDS (with caveat)

**Premise**: `user === username && pass === password` uses JavaScript `===` comparison.

**Mechanism**: middleware.ts:34 — string equality comparison short-circuits on first differing character.

**Challenge**: "What evidence would prove this check doesn't actually protect?" — An attacker would need to measure sub-millisecond response time differences over a network. Network jitter typically dwarfs string comparison timing. For a local SQLite app primarily designed for local use, this is low-risk.

**Verdict**: HOLDS. The timing leak exists technically, but the threat model (local/personal tool, Basic Auth as optional layer) makes exploitation impractical. A constant-time comparison would be better practice but this isn't a critical vulnerability in context.

---

### S3: Bookmarklet unauthenticated write — VULNERABLE (same root as S1)

Subsumed by S1. The endpoint's CORS headers are correctly restrictive for browser contexts, but the server-side lack of auth is the real issue. See S1.

---

### S4: Unbounded Twitter API pagination — VULNERABLE

**Premise**: "We'll keep fetching pages until Twitter returns no more results or no next cursor."

**Mechanism**: app/api/import/twitter/route.ts:270 — `while (true)` with break on `!nextCursor || tweets.length === 0`. No page counter, no timeout.

**Forward**: If a user has 50,000+ bookmarks, or if the Twitter API returns cursors indefinitely (malformed response, API change), this loop never terminates. The HTTP request handler blocks indefinitely.

**Verify**: Compare with lib/x-sync.ts:17-41 — the sync function HAS `MAX_PAGES = 50` protection. The twitter import route does NOT have this guard. This is inconsistent — same operation, different safety boundaries.

**WHY**:
- **L1**: The developer assumed the Twitter API would naturally terminate pagination. The sync path was written later and added the guard, but the import route was not updated.
- **L2**: Meta-pattern: "Safety guard applied to one path but not all paths performing the same operation."

**ABSTRACT**: Search for other unbounded loops.

Checked: lib/categorizer.ts uses cursor-based pagination with `BATCH_SIZE` and proper breaks. lib/rawjson-extractor.ts has `CHUNK` with proper breaks. lib/vision-analyzer.ts has chunk-based pagination. Only the twitter import route lacks bounds.

---

### S5 + S6: Hardcoded and divergent Bearer tokens — VULNERABLE

**Premise**: "The Bearer token is a public, well-known Twitter app token used by all third-party clients."

**Mechanism**: Two files, two different Bearer tokens:
- lib/twitter-api.ts:5 — `AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA`
- app/api/import/twitter/route.ts:4 — `AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I%2BxMb1nYFAA%3DUognEfK4ZPxYowpr4nMskopkC%2FDO`

**Effect**: The divergence means one path uses a different app identity than the other. If Twitter rotates/revokes one token, only one import path breaks — silent inconsistency. More critically, having two copies of functionality (parsePage, extractMedia, tweetFullText, bestVideoUrl duplicated between lib/twitter-api.ts and app/api/import/twitter/route.ts) means bug fixes in one don't propagate to the other.

**WHY**:
- **L1**: Code was duplicated between the API route and the library instead of importing shared functions.
- **L2**: Meta-pattern: "Duplicated logic with drift" — same algorithm copy-pasted across files, diverging silently over time.

**ABSTRACT**: Grep for other duplicated functions.

**Found**: `tweetFullText`, `extractMedia`, `bestVideoUrl`, `parsePage`, `decodeHtmlEntities` are all duplicated across:
1. lib/twitter-api.ts
2. app/api/import/twitter/route.ts
3. app/api/import/bookmarklet/route.ts (extractMedia, tweetFullText, bestVideoUrl)

Three copies of the same parsing logic. bookmarklet/route.ts:67-79 `tweetFullText` is MISSING the `decodeHtmlEntities` call that the other two have — concrete divergence bug.

---

### S7: $executeRawUnsafe with FTS_TABLE — HOLDS

**Premise**: The FTS_TABLE name is a hardcoded constant `'bookmark_fts'`.

**Mechanism**: lib/fts.ts:11 — `const FTS_TABLE = 'bookmark_fts'` never changes, never comes from user input.

**Challenge**: Could any code path change FTS_TABLE? No — it's a file-level `const`, not exported, not derived from any input.

**Verdict**: HOLDS. The `$executeRawUnsafe` is used with a constant string. No injection vector exists. The parameterized `$executeRaw` is correctly used for the INSERT with user data (line 56-59).

---

### S8: FTS5 MATCH query injection — VULNERABLE

**Premise**: "We sanitize FTS5 special chars by removing `\"*()` and that's sufficient."

**Mechanism**: lib/fts.ts:78 — `kw.replace(/["*()]/g, ' ')` then terms joined with ` OR `.

**Forward**: FTS5 syntax includes operators not stripped: `^` (start-of-column), `NEAR`, `NOT`, column-prefix syntax (`text:`, `semantic_tags:`). A search term like `text:password OR entities:secret` would be passed through to the MATCH clause, allowing column-targeted searches. More critically, a bare `*` is stripped but `NOT *` would become `NOT` which is a valid FTS5 operator.

**Verify**: The ftsSearch function is called from two paths:
1. app/api/search/ai/route.ts:231 — keywords from `extractKeywords()` which strips non-alphanumeric chars (lib/search-utils.ts:13: `replace(/[^a-z0-9\s]/g, ' ')`) — this path is SAFE because alphanumeric-only keywords can't contain FTS5 operators.
2. cli/siftly.ts:53 — keywords from `extractKeywords()` — same sanitization, SAFE.

**Verdict**: HOLDS (in current call paths). The FTS5 sanitization in fts.ts is incomplete, but all current callers pre-sanitize keywords via `extractKeywords()` which strips all non-alphanumeric characters. However, if any future caller passes raw user input to `ftsSearch` without going through `extractKeywords`, the injection would be exploitable. This is a latent vulnerability (defense exists but is in the wrong layer).

---

### S9: API keys in plaintext SQLite — HOLDS (by design, acknowledged)

**Premise**: "This is a local/self-hosted tool."

**Mechanism**: API keys are stored via `prisma.setting.upsert` in the Settings table. The app acknowledges this: app/settings/page.tsx:626 explicitly warns "Keys are stored in plaintext in your local SQLite database."

**Verdict**: HOLDS. Acknowledged design decision for a local tool. The warning exists. However, see S10 for an unintended credential injection path.

---

### S10: API key injection from categorize request body — VULNERABLE

**Premise**: "The user can pass an API key with the categorize request for convenience."

**Mechanism**: app/api/categorize/route.ts:112-119:
```typescript
if (apiKey && typeof apiKey === 'string' && apiKey.trim() !== '') {
    const currentProvider = await getProvider()
    const keySlot = currentProvider === 'openai' ? 'openaiApiKey' : 'anthropicApiKey'
    await prisma.setting.upsert({
      where: { key: keySlot },
      update: { value: apiKey.trim() },
      create: { key: keySlot, value: apiKey.trim() },
    })
}
```

**Effect**: Any authenticated client (or unauthenticated if no auth is configured — the default) can overwrite the stored API key by POSTing to `/api/categorize` with an `apiKey` field. This is a side-effect of the categorize endpoint, not documented as a settings endpoint.

**WHY**:
- **L1**: Convenience feature — let the user set their key from the categorize UI without going to settings first.
- **L2**: But this means the categorize endpoint is a covert settings-write endpoint. If auth is misconfigured or absent (default), anyone can replace the API key with their own (to capture prompts) or with an invalid key (DoS).
- **L3**: Meta-pattern: "Side-effect credential write in non-settings endpoint" — privileged operation hidden in a non-privileged path.

---

### S11: Stale settings cache — HOLDS

**Premise**: 5-minute TTL cache for provider and model settings.

**Mechanism**: lib/settings.ts:59-66 — `invalidateSettingsCache()` is called by app/api/settings/route.ts after every settings change.

**Verify**: Grep confirms `invalidateSettingsCache` is called on every POST to settings. The only scenario where staleness matters is if settings are changed via direct DB manipulation (bypassing the API), which is an edge case.

**Verdict**: HOLDS. Cache invalidation is properly wired to the settings API.

---

### S12: Module-level syncing flag — HOLDS (with caveat)

**Premise**: `let syncing = false` in lib/x-sync.ts protects against concurrent syncs.

**Challenge**: In a single Node.js process (standard Next.js deployment), this is safe because JavaScript is single-threaded. Multiple processes (cluster mode, multiple containers) would not share this flag. However, SQLite itself constrains concurrent writes, so the worst case is a duplicate sync attempt that fails on DB write contention.

**Verdict**: HOLDS for the stated deployment model (single-process, local tool). Would break under multi-process deployment.

---

### S13: Twitter credentials in request body — HOLDS (by design, with S14 caveat)

**Premise**: The twitter import route accepts authToken and ct0 directly in the POST body.

**Mechanism**: This is a one-time import flow — the user pastes their browser cookies. The credentials are used immediately and not stored by this route.

**Verdict**: HOLDS for the import route. But see S14 for the live-sync path where they ARE stored.

---

### S14: X auth credentials stored in plaintext — VULNERABLE

**Premise**: "We need to store auth_token and ct0 for scheduled sync."

**Mechanism**: app/api/import/live/route.ts:58-71 — X session tokens (auth_token, ct0) are stored in the Settings table, same as API keys.

**Effect**: These are session cookies for the user's X/Twitter account. Unlike API keys (which the user explicitly creates for this purpose), these are session tokens that grant full account access. Storing them in plaintext SQLite means anyone with file access to `dev.db` gets the user's full X session.

**WHY**:
- **L1**: Same storage pattern as API keys — "all secrets go in Settings."
- **L2**: Meta-pattern: "Uniform secret storage without risk tiering" — API keys and session tokens have very different blast radii but are stored identically.

---

### S15: Vision CONCURRENCY = 12 — HOLDS

**Premise**: High concurrency for parallel vision API calls.

**Challenge**: Could this exhaust rate limits? The retry logic in vision-analyzer.ts:139-165 handles 429/overloaded/529 responses with exponential backoff. The concurrency is bounded (not infinite), and the CHUNK size (15) limits total inflight.

**Verdict**: HOLDS. The retry/backoff mechanism handles rate-limiting gracefully. 12 concurrent requests is aggressive but bounded.

---

### S16: Bookmark ID not validated before category delete — VULNERABLE

**Premise**: The PUT handler trusts that the `id` parameter is a valid bookmark.

**Mechanism**: app/api/bookmarks/[id]/categories/route.ts:21 — `prisma.bookmarkCategory.deleteMany({ where: { bookmarkId: id } })` — if `id` doesn't match any bookmark, this is a no-op (not harmful). But then line 25 `prisma.bookmarkCategory.createMany` will try to create records with a non-existent bookmarkId, which will fail with a foreign key constraint error since the schema has `onDelete: Cascade` on the relation.

**Verify**: Actually, Prisma with SQLite and the schema's FK constraint would throw on createMany with an invalid bookmarkId. This would be caught by the catch block (line 35) and return a 500 error.

**Verdict**: HOLDS (low severity). The operation is a no-op for delete and fails safely on create. The error handling catches it. Missing 404 is a UX issue, not a security/correctness issue.

---

### S17: Greedy regex for JSON extraction — HOLDS (with caveat)

**Premise**: `text.match(/\[[\s\S]*\]/)` extracts the JSON array from AI response.

**Challenge**: If the AI response contains text like `[intro text] ... [{"tweetId": ...}]`, the regex would match from the first `[` to the last `]`, including non-JSON content. However, the AI prompt explicitly says "Return ONLY valid JSON — no markdown, no explanation." The JSON.parse call immediately after would throw on malformed content, and errors are caught.

**Verdict**: HOLDS. The regex is greedy but the JSON.parse validation catches bad extractions. Failed parses are handled (console.warn + fallback).

---

### S18: CORS default to x.com — HOLDS

**Premise**: When the origin isn't in the allowed set, the response defaults to `https://x.com` rather than rejecting.

**Mechanism**: app/api/import/bookmarklet/route.ts:8 — `const allowed = ALLOWED_ORIGINS.has(origin) ? origin : 'https://x.com'`

**Challenge**: This means a request from `https://evil.com` gets a response with `Access-Control-Allow-Origin: https://x.com`. The browser at evil.com sees this doesn't match and blocks the response. This is correct — the response is blocked by the browser because the origin doesn't match.

**Verdict**: HOLDS. The fallback to x.com doesn't grant access to non-whitelisted origins. Browser CORS enforcement is correct here.

---

### S19: SSRF — incomplete private IP check — VULNERABLE

**Premise**: `isPrivateUrl` blocks common private IP ranges and protocols.

**Mechanism**: app/api/link-preview/route.ts:11-29 — Checks IPv4 private ranges, localhost, `::1`, `fd*` ULA. Post-redirect check on line 163.

**Missing**:
1. IPv6-mapped IPv4: `::ffff:127.0.0.1` — not blocked (regex only checks `::1` and `fd*`)
2. `0.0.0.0` is blocked, but decimal IP encoding (`2130706433` = 127.0.0.1) not checked
3. DNS rebinding: first resolution passes the check, but the actual fetch could resolve to a different IP
4. `[::]` (IPv6 unspecified) not explicitly blocked

**WHY**:
- **L1**: The developer covered the most common private ranges but IPv6 edge cases were missed.
- **L2**: Meta-pattern: "Blocklist-based SSRF protection" — blocklists are inherently incomplete; an allowlist approach (only allowing http/https to verified public IPs) would be more robust.

**Verify**: The post-redirect check on line 163 `isPrivateUrl(res.url)` re-checks the final URL, which is good. But DNS rebinding affects the connection itself, not the URL.

---

### S20: No rate limiting — VULNERABLE

**Premise**: Implicit assumption that the app is used locally by a single user.

**Mechanism**: No rate limiting middleware, no throttling on any endpoint. Not in middleware.ts, not in individual route handlers.

**Forward**: If the instance is exposed to the internet (which is why Basic Auth exists as an option):
- /api/categorize: starts an expensive AI pipeline (vision + enrichment + categorization) — one request can trigger hundreds of API calls
- /api/search/ai: each request makes an AI API call — can burn through API quota
- /api/import/twitter: triggers unbounded Twitter API calls (S4)
- /api/link-preview: can be used as an SSRF proxy (partially mitigated by S19 checks)

**WHY**:
- **L1**: "It's a personal tool, only I use it."
- **L2**: But the auth system exists precisely for the case where it's exposed. If it's exposed, rate limiting is needed.
- **L3**: Meta-pattern: "Auth without rate limiting" — authentication controls WHO can access, rate limiting controls HOW MUCH they can use. They serve different purposes.

---

### S21: Fire-and-forget async pipeline — HOLDS (with caveat)

**Premise**: `void (async () => { ... })()` runs the pipeline in the background.

**Mechanism**: app/api/categorize/route.ts:152 — The async IIFE is not awaited. The `.then()` and `.catch()` handlers on lines 371-390 handle completion and errors.

**Verify**: The `.catch()` at the end does handle rejections by setting the error state. Unhandled rejection would only occur if the `.catch()` itself threw, which is unlikely (it only calls setState and console.error). The inner try/catch (line 155) also catches pipeline errors.

**Verdict**: HOLDS. Error handling is present at both the inner and outer levels. The fire-and-forget pattern is appropriate for a long-running background pipeline that reports progress via GET.

---

### S22: rebuildFts loads all bookmarks — VULNERABLE

**Premise**: "FTS rebuild is a batch operation, load everything then insert."

**Mechanism**: lib/fts.ts:34-42 — `prisma.bookmark.findMany({ select: ... })` with NO pagination — loads all bookmarks into memory.

**Forward**: For a user with 100,000 bookmarks, this loads all text, semanticTags, entities, and imageTags into a single Node.js array. Each bookmark could have 1-5KB of text+tags, so 100K bookmarks = 100-500MB in memory. Node.js default heap is ~1.5GB. This could cause OOM for large libraries.

**Note**: The INSERT is properly batched (200 per transaction, line 47-48), but the SELECT is not.

**WHY**:
- **L1**: Simple implementation — load all, then batch-insert.
- **L2**: Meta-pattern: "Batched writes but unbatched reads" — the developer thought about write batching but not read batching.

---

### S23: REPLACE_ME placeholder in production — VULNERABLE

**Premise**: The Likes query ID is a placeholder.

**Mechanism**: app/api/import/twitter/route.ts:44 — `queryId: 'REPLACE_ME'`

**Effect**: If a user attempts to import likes (source='like'), the request will hit Twitter's API with an invalid query ID, resulting in a 400 error. The error is caught (line 321-326) and returned as a 500 to the user with a cryptic error message from Twitter. No graceful error explaining the feature is unconfigured.

**Verdict**: VULNERABLE — not a security issue but a broken feature shipped as-if-working. The UI presumably shows a "likes import" option that simply fails.

---

### S24: Subdomain matching false positives — HOLDS (low severity)

**Premise**: `domain.endsWith(knownDomain)` for tool detection.

**Mechanism**: lib/rawjson-extractor.ts:148 — `domain.endsWith(knownDomain)`.

**Example**: `notgithub.com`.endsWith(`github.com`) = true. This would incorrectly tag a tweet as linking to "GitHub".

**Impact**: This only affects the `tools` field in entity extraction, which feeds into AI categorization hints. A false positive tool detection would slightly bias categorization but the AI prompt already uses multiple signals. Minimal real-world impact.

**Verdict**: HOLDS (low severity). The false positive rate is negligible — real-world URLs with these patterns are extremely rare.

---

### S25: Static media proxy allowlist — HOLDS (by design)

**Premise**: Only Twitter media CDN hosts are allowed.

**Mechanism**: app/api/media/route.ts:3-9 — hardcoded set of twimg.com domains.

**Challenge**: Article cover images hosted elsewhere would fail. But the media proxy is specifically for Twitter/X media, not a general proxy. Article images are loaded directly by the browser via img src.

**Verdict**: HOLDS. The allowlist correctly restricts the proxy to its intended purpose.

---

### S26: All API routes require Basic Auth — HOLDS

**Premise**: The middleware matcher covers everything except Next.js internals.

**Mechanism**: middleware.ts:49-54 — regex excludes `_next/`, `favicon.ico`, `icon.svg`. All other paths including `/api/*` require auth.

**Verify**: The only exception is `/api/import/bookmarklet` (S1). All other API routes correctly require auth when auth is configured.

**Verdict**: HOLDS (except S1).

---

## ABSTRACT-Discovered Seeds

### A1: Code duplication divergence (from S5/S6 ABSTRACT)

**Discovery**: Three copies of Twitter parsing logic with concrete drift.

**Files**:
- lib/twitter-api.ts — canonical implementation with `decodeHtmlEntities`
- app/api/import/twitter/route.ts — separate copy with different Bearer token, has `decodeHtmlEntities`
- app/api/import/bookmarklet/route.ts — third copy, `tweetFullText` MISSING `decodeHtmlEntities` call

**Concrete bug**: bookmarklet/route.ts:67-79 — `tweetFullText` returns raw HTML entities (`&amp;`, `&lt;`, etc.) while the other two implementations decode them. Bookmarks imported via bookmarklet will have HTML entities in their text, affecting search, display, and AI categorization quality.

**Verdict**: VULNERABLE.

### A2: Settings cache invalidation not called from categorize route (from S10/S11 ABSTRACT)

**Discovery**: When the categorize route writes an API key (S10), it does NOT call `invalidateSettingsCache()`. The old key remains cached for up to 5 minutes after the new one is written.

**Verify**: app/api/categorize/route.ts:112-119 — writes to settings via prisma.setting.upsert but does not import or call `invalidateSettingsCache` from lib/settings.ts.

app/api/settings/route.ts correctly calls `invalidateSettingsCache()` after every setting change.

**Verdict**: VULNERABLE. After the key is overwritten by the categorize endpoint, the pipeline continues using the cached (old) key from settings.ts for up to 5 minutes.

---

## Output

### Seed Map Summary

- **Total seeds**: 26 initial + 2 ABSTRACT = 28
- **VULNERABLE**: 12 (S1, S3, S4, S5, S6, S10, S14, S19, S20, S22, S23, A1, A2 — S3 merged with S1)
- **HOLDS**: 13 (S2, S7, S8, S9, S11, S12, S13, S15, S16, S17, S18, S24, S25, S26 — some with caveats)
- **UNCLEAR**: 0

### Root Causes (grouped by shared failed assumption)

**RC1: CORS-as-authentication (S1/S3)**
- Failed assumption: "Browser-enforced origin restrictions protect server endpoints."
- Meta-pattern: Relying on client-side enforcement for server-side security.
- Affected: /api/import/bookmarklet — unauthenticated write path when auth is configured.

**RC2: Duplicated logic with silent drift (S5, S6, A1)**
- Failed assumption: "Copy-pasting the parsing code into each route is fine, they do the same thing."
- Meta-pattern: Code duplication creates maintenance divergence — bug fixes and improvements in one copy don't propagate.
- Affected: Three copies of Twitter parsing logic (tweetFullText, extractMedia, bestVideoUrl, parsePage), two different Bearer tokens, bookmarklet missing HTML entity decoding.

**RC3: Uniform secret storage without risk tiering (S9, S10, S14)**
- Failed assumption: "All secrets can use the same storage mechanism."
- Meta-pattern: API keys (user-created, scoped) and session tokens (full account access) have different blast radii but identical storage/handling.
- Affected: X session tokens in plaintext SQLite, API key injection from non-settings endpoint.

**RC4: Safety guards applied inconsistently (S4, S22)**
- Failed assumption: "If one path has the guard, the system is safe."
- Meta-pattern: Same operation implemented in multiple code paths, safety bounds applied to only some.
- Affected: lib/x-sync.ts has MAX_PAGES=50, app/api/import/twitter/route.ts has no bound. lib/fts.ts batches writes but not reads.

**RC5: Auth without operational controls (S20)**
- Failed assumption: "Authentication is sufficient for exposed deployments."
- Meta-pattern: Authentication controls access identity, not access volume. Rate limiting is a separate concern.
- Affected: All endpoints — no rate limiting, no request size limits.

### Assumption Radiation Map

```
RC2 (Code Duplication)
├── S5: Hardcoded Bearer token (lib/twitter-api.ts:5)
├── S6: Different Bearer token (app/api/import/twitter/route.ts:4)
├── A1 [ABSTRACT]: bookmarklet missing decodeHtmlEntities
└── drift risk: any future bugfix to one copy won't reach others

RC1 (CORS-as-auth)
├── S1: middleware.ts:21 bookmarklet bypass
└── S3: bookmarklet/route.ts — unauthenticated POST

RC3 (Uniform secret storage)
├── S9: API keys in plaintext (acknowledged)
├── S10: categorize route writes API key as side-effect
├── S14: X session tokens in plaintext
└── A2 [ABSTRACT]: cache not invalidated after categorize key write

RC4 (Inconsistent guards)
├── S4: twitter import unbounded loop
└── S22: FTS rebuild unbatched read

RC5 (Auth without ops)
└── S20: No rate limiting on any endpoint
```

### Impact Chains

1. **S1 → data injection**: Attacker hits `/api/import/bookmarklet` (no auth) → injects bookmarks with malicious URLs → user clicks link in UI → phishing/malware.

2. **S4 → resource exhaustion**: User triggers twitter import with large account → unbounded pagination → request hangs → Node.js event loop blocked → entire app unresponsive.

3. **S10 → A2 → API key hijack**: Attacker (or misconfigured client) POSTs to `/api/categorize` with their own API key → key saved to DB → but pipeline continues using cached old key for 5 min → eventual switch to attacker's key → all AI prompts (containing bookmark text) sent through attacker's API key endpoint.

4. **A1 → data quality**: Bookmarklet imports store raw HTML entities in text → FTS search misses terms with `&amp;` → AI categorization sees malformed text → lower categorization quality for bookmarklet-imported data.

5. **S22 → OOM**: Large library → FTS rebuild loads all bookmarks → Node.js heap exhaustion → process crash → data loss if mid-transaction.

### Defense Map

| Defense | Coverage | Gaps |
|---------|----------|------|
| Basic Auth (middleware.ts) | All routes except /api/import/bookmarklet | S1: bookmarklet bypass; no rate limiting (S20) |
| CORS origin whitelist (bookmarklet) | Browser-originated requests | Non-browser clients ignore CORS (S1) |
| SSRF blocklist (link-preview) | Common private IPv4 ranges | IPv6-mapped IPv4, DNS rebinding (S19) |
| Media proxy allowlist | twimg.com domains only | Correctly scoped |
| FTS sanitization (fts.ts) | Strips `"*()` | Incomplete for FTS5 syntax, but callers pre-sanitize via extractKeywords (S8) |
| Settings cache invalidation | Called on all settings route changes | Not called from categorize route key write (A2) |
| Sync concurrency guard | Module-level boolean | Single-process only (S12) |
| AI response validation | JSON.parse after regex extraction | Greedy regex mitigated by parse validation (S17) |
| Max pagination (x-sync) | MAX_PAGES=50 for scheduled sync | NOT applied to twitter import route (S4) |

### Unexplored

None — all seeds received verdicts.
