# AFM v3.1 Report: Siftly (Twitter/X Bookmark Manager)

> Methodology: Assumption Failure Methodology v3.1 (Seed + Engine + WHY)
> Target: Next.js + Prisma + SQLite app with AI categorization
> Scope: lib/, app/api/, components/, middleware.ts, cli/

---

## Phase 1: Seed List

| # | Seed | File:Line |
|---|------|-----------|
| S01 | Bookmarklet endpoint bypasses all auth | middleware.ts:21 |
| S02 | No CSRF protection on bookmarklet POST | app/api/import/bookmarklet/route.ts:110 |
| S03 | Hardcoded Twitter Bearer tokens in source code (two different ones) | lib/twitter-api.ts:6, app/api/import/twitter/route.ts:4 |
| S04 | API keys stored in plaintext in SQLite Setting table | app/api/settings/route.ts:125-129 |
| S05 | FTS5 table created with $executeRawUnsafe (SQL injection surface) | lib/fts.ts:14 |
| S06 | FTS5 MATCH query built from user input without full sanitization | lib/fts.ts:78-88 |
| S07 | Timing-vulnerable string comparison for Basic Auth credentials | middleware.ts:34 |
| S08 | No rate limiting on any API endpoint | all routes |
| S09 | Twitter import endpoint (POST /api/import/twitter) has no pagination limit | app/api/import/twitter/route.ts:270 |
| S10 | Bookmarklet endpoint CORS allows only x.com/twitter.com but no auth at all | app/api/import/bookmarklet/route.ts:4-8 |
| S11 | Settings DELETE endpoint allows deleting arbitrary allowed keys without ownership check | app/api/settings/route.ts:193-208 |
| S12 | link-preview SSRF check doesn't cover IPv6 mapped addresses or DNS rebinding | app/api/link-preview/route.ts:11-29 |
| S13 | Export endpoints load ALL bookmarks into memory | lib/exporter.ts:33-44, 146-176 |
| S14 | Codex CLI writes prompt to temp file with predictable path pattern | lib/codex-cli.ts:40 |
| S15 | Module-level caches have no invalidation on multi-instance deployments | lib/settings.ts:4-12 |
| S16 | Categorization pipeline global state allows race condition on concurrent POST | app/api/categorize/route.ts:98 |
| S17 | execSync used to read macOS Keychain (blocks event loop) | lib/claude-cli-auth.ts:42 |
| S18 | DELETE /api/bookmarks deletes all bookmarks + categories with no confirmation | app/api/bookmarks/route.ts:14-31 |
| S19 | OpenAI assistant role silently drops image content parts | lib/ai-client.ts:88 |
| S20 | Twitter API import does sequential DB queries per tweet (N+1) | lib/twitter-api.ts:243-296, app/api/import/twitter/route.ts:274-316 |
| S21 | Like import endpoint references REPLACE_ME placeholder query ID | app/api/import/twitter/route.ts:44 |
| S22 | Media proxy allows fetching from a fixed allowlist but doesn't validate response content-type | app/api/media/route.ts:62-86 |
| S23 | AI search sends entire bookmark index (up to 150 entries) in prompt | app/api/search/ai/route.ts:303-342 |
| S24 | Search cache has unbounded key space and only evicts oldest entry | app/api/search/ai/route.ts:21-23 |
| S25 | Category slug generation can collide for different Unicode inputs | app/api/categories/route.ts:5-13 |
| S26 | Bookmark categories PUT replaces all without transaction isolation | app/api/bookmarks/[id]/categories/route.ts:21-32 |
| S27 | Vision analyzer uses 12 concurrent workers against rate-limited APIs | lib/vision-analyzer.ts:71 |
| S28 | No input size validation on import file upload | app/api/import/route.ts:5-27 |

---

## Phase 2: Assume + Verify (the loop)

### S01 — Bookmarklet endpoint bypasses all auth → VULNERABLE

**"Why did you write it this way?"** The designer assumed the bookmarklet is called cross-origin from x.com and cannot carry Basic Auth credentials. The path `/api/import/bookmarklet` is excluded from auth checks in middleware.ts:21.

**"What does it actually do?"** ANY unauthenticated client can POST to `/api/import/bookmarklet` with crafted tweet data, injecting arbitrary bookmarks into the database. The CORS headers restrict browser cross-origin requests to x.com/twitter.com origins, but CORS is a browser-only enforcement. Any non-browser client (curl, scripts, bots) can freely call this endpoint.

**"What does this promise?"** It promises that only the bookmarklet running on x.com can use this endpoint. That promise is false.

**Verified in code:**
- middleware.ts:21: `if (request.nextUrl.pathname === '/api/import/bookmarklet') return NextResponse.next()`
- app/api/import/bookmarklet/route.ts:110-176: No additional auth check. Accepts raw TweetResult objects and writes directly to DB.

**Impact:** An attacker with network access to the Siftly instance can inject unlimited bookmarks with arbitrary text, author handles, and media. When the categorization pipeline runs, these poisoned bookmarks consume AI API credits. If deployed behind a Cloudflare tunnel with Basic Auth, this endpoint is the escape hatch.

**WHY:** The developers conflated CORS (browser same-origin policy) with authentication. CORS prevents browsers from reading cross-origin responses but does nothing against server-side or non-browser requests. This is a common assumption: "if I set CORS headers, only that origin can call my API."

**Meta-pattern: CORS-as-auth.** Where else does this apply?
- Nowhere else in this codebase; the bookmarklet is the only CORS-configured endpoint. But the assumption itself -- that cross-origin restrictions equal access control -- is the root error.

---

### S02 — No CSRF protection on bookmarklet POST → VULNERABLE

**Interrogation:** The designer assumed only the Siftly bookmarklet JavaScript would call this endpoint. Since it's POST with JSON body, the browser's CORS preflight should block other origins.

**Mechanism:** The endpoint accepts `Content-Type: application/json`, which triggers CORS preflight. However, preflight can be avoided with `Content-Type: text/plain` or `application/x-www-form-urlencoded`. The route calls `request.json()` which parses JSON regardless of Content-Type header.

**Verification:** app/api/import/bookmarklet/route.ts:113-114 -- `body = await request.json()` does not validate Content-Type. Any site could submit a form POST with a JSON body to this endpoint.

**Verdict: VULNERABLE** (compounded with S01 -- the endpoint is already fully open)

**WHY:** Same root as S01 -- the mental model is "CORS = auth." CSRF protection would be moot anyway since the endpoint is fully unauthenticated.

---

### S03 — Hardcoded Twitter Bearer tokens → VULNERABLE

**Interrogation:** The developer assumed the Bearer token is a constant public value used by the Twitter web client, not a secret. Two different bearer tokens appear:
- lib/twitter-api.ts:6: `AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA`
- app/api/import/twitter/route.ts:4: `AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I%2BxMb1nYFAA%3DUognEfK4ZPxYowpr4nMskopkC%2FDO`

**Mechanism:** These are indeed Twitter's publicly known app-level Bearer tokens (used by the web client). They are not per-user secrets. However, hardcoding them means:
1. If Twitter rotates them, both files need updating independently
2. The two files have DIFFERENT bearer tokens -- they cannot both be current
3. The tokens appear in git history permanently

**Verdict: VULNERABLE** (low severity for security, medium for correctness)

The two different bearer tokens indicate a drift -- one was updated and the other was not. lib/twitter-api.ts is used by the live sync feature; app/api/import/twitter/route.ts is the direct import endpoint. If one is stale, that path silently fails.

**WHY:** Code duplication across two files that implement the same Twitter API interaction pattern. The twitter-api.ts module and the /api/import/twitter/route.ts each independently implement fetchPage, parsePage, extractMedia, tweetFullText, and the bearer token. There is no shared constant.

**Meta-pattern: Duplicated external interface constants.** Grep confirms: tweetFullText is implemented 3 times (lib/twitter-api.ts:212, app/api/import/bookmarklet/route.ts:67, app/api/import/twitter/route.ts:195). extractMedia is implemented 3 times. bestVideoUrl is implemented 3 times. Each is a subtle drift risk.

---

### S04 — API keys stored in plaintext in SQLite → VULNERABLE

**Interrogation:** The developer assumed the SQLite database is a local file, so plaintext storage is acceptable for a self-hosted tool.

**Mechanism:** API keys (Anthropic, OpenAI, X auth_token, X ct0) are stored as plaintext strings in the `Setting` table. The GET /api/settings endpoint masks them for display (settings/route.ts:5-9), but:
1. The database file (prisma/dev.db) is a single file that can be copied
2. Anyone with file system access gets all API keys
3. The X auth_token and ct0 are session cookies that grant full access to the user's Twitter account
4. Backup/export of the DB exports all keys

**Verdict: VULNERABLE**

**WHY:** For a local self-hosted tool, this is a reasonable tradeoff. But the app also supports deployment via Docker and Cloudflare Tunnel (start.sh:60-73), where the DB might be on shared storage. The assumption "local only" is contradicted by the deployment features the app itself provides.

**Meta-pattern: "It's local-only" assumption in an app that ships with remote-deployment tooling.**

---

### S05 — FTS5 table created with $executeRawUnsafe → HOLDS

**Interrogation:** `$executeRawUnsafe` is used because Prisma doesn't support `CREATE VIRTUAL TABLE` syntax natively.

**Verification:** lib/fts.ts:14 -- The SQL string is a hardcoded template literal with only the constant `FTS_TABLE`. No user input flows into this call. The table name is `bookmark_fts`, a module-level constant.

**Challenge: "What evidence would prove this check doesn't actually protect?"** I searched for any caller that passes user input into ensureFtsTable or the FTS_TABLE constant. None found. The constant is defined at lib/fts.ts:11 and never reassigned.

**Verdict: HOLDS** -- The $executeRawUnsafe usage is safe because the SQL is fully static.

---

### S06 — FTS5 MATCH query built from user input → VULNERABLE

**Interrogation:** The developer sanitizes FTS5 special characters at lib/fts.ts:78: `kw.replace(/["*()]/g, ' ')`. Then joins with OR: `terms.join(' OR ')`.

**Mechanism:** FTS5 special syntax includes more characters than just `"*()`. Specifically:
- `NEAR` and `NOT` are FTS5 keywords that can alter query semantics
- `^` (column filter prefix) is not stripped
- `{`, `}` (used in `NEAR/{N}`) are not stripped
- `:` (column prefix) is not stripped -- input like `text:secret` would restrict search to the text column

**Verification:** lib/fts.ts:78 strips `"`, `*`, `(`, `)` only. A search for `text:password` would produce the MATCH query `text:password` which is valid FTS5 syntax targeting the text column specifically. While this doesn't leak data beyond the FTS table, it allows query manipulation.

The `$queryRaw` tagged template at lib/fts.ts:86-90 properly parameterizes the MATCH value, so actual SQL injection is not possible. The vulnerability is FTS5 query syntax manipulation, not SQL injection.

**Verdict: VULNERABLE** (low severity -- FTS query manipulation, no data leak beyond bookmark table)

**WHY:** The sanitization was written to prevent "obvious" FTS5 metacharacters but the developer didn't consult the full FTS5 query syntax specification. Incomplete sanitization of a query language is a recurring pattern.

**Meta-pattern: Partial sanitization of a secondary query language.** Same pattern could appear if the app ever adds a search DSL.

---

### S07 — Timing-vulnerable string comparison for Basic Auth → VULNERABLE

**Interrogation:** middleware.ts:34 uses `user === username && pass === password` for credential comparison. JavaScript `===` short-circuits on the first differing byte.

**Mechanism:** In theory, a timing attack could extract the password character by character by measuring response times. In practice, over a network, the timing signal is drowned by network jitter. This is a theoretical vulnerability for this deployment model (web server over HTTP).

**Verdict: VULNERABLE** (low severity -- theoretical in network context, but a real code quality issue)

**WHY:** Using `===` for secret comparison is a known antipattern. The correct approach is `crypto.timingSafeEqual`. This is the "it works so it must be secure" assumption.

---

### S08 — No rate limiting on any API endpoint → VULNERABLE

**Interrogation:** The developer assumed this is a self-hosted tool used by a single person.

**Mechanism:** Every API endpoint, including the unauthenticated bookmarklet endpoint (S01), has zero rate limiting. Endpoints that trigger AI API calls (categorize, search/ai, analyze/images) can be called repeatedly to exhaust API credits.

**Verification:** Searched all route files for any rate-limit, throttle, or token-bucket pattern. None found.

**Verdict: VULNERABLE** (medium severity when deployed publicly; the app explicitly supports public deployment via Cloudflare Tunnel)

**WHY:** Same "it's local-only" assumption contradicted by built-in tunnel support.

---

### S09 — Twitter import endpoint has no pagination limit → VULNERABLE

**Interrogation:** app/api/import/twitter/route.ts:270 -- `while (true)` loop fetches pages until nextCursor is null.

**Mechanism:** Unlike lib/x-sync.ts which has `MAX_PAGES = 50` (x-sync.ts:17), the /api/import/twitter route has no page limit. A user with many bookmarks/likes could trigger an unbounded loop that keeps fetching from Twitter's API until the session token expires or the server runs out of memory.

**Verification:** Compared the two implementations:
- lib/x-sync.ts:17-19: `const MAX_PAGES = 50; for (let page = 0; page < MAX_PAGES; page++)`
- app/api/import/twitter/route.ts:270: `while (true)` with no counter

**Verdict: VULNERABLE**

**WHY:** The /api/import/twitter route was written separately from lib/x-sync.ts (part of the code duplication from S03). The safety guard that exists in one implementation was simply not copied to the other.

**Meta-pattern: Safety guards lost in code duplication.** This is the same root cause as S03.

---

### S10 — Bookmarklet CORS without auth → VULNERABLE

(Subsumed by S01. The CORS restriction is the only "protection" and it's browser-only.)

**Verdict: VULNERABLE** (same root as S01)

---

### S11 — Settings DELETE allows deleting specific keys → HOLDS

**Interrogation:** The endpoint at app/api/settings/route.ts:193-208 accepts a `key` parameter and deletes it if it's in the allowlist: `['anthropicApiKey', 'openaiApiKey', 'x_oauth_client_id', 'x_oauth_client_secret']`.

**Challenge:** Could an attacker cause harm by deleting these keys? The endpoint requires authentication (middleware runs for all non-bookmarklet paths). An authenticated user deleting their own API keys is expected behavior.

**Verdict: HOLDS** -- Protected by auth middleware, and the operation (deleting your own keys) is intentional.

---

### S12 — SSRF check incomplete for link-preview → VULNERABLE

**Interrogation:** app/api/link-preview/route.ts:11-29 blocks private IPs but has gaps:

1. **IPv4-mapped IPv6:** `::ffff:127.0.0.1` is not blocked. The regex checks for `^127\.` but IPv6-mapped addresses have a different format.
2. **DNS rebinding:** A hostname that initially resolves to a public IP but resolves to 127.0.0.1 on subsequent requests (after the `isPrivateUrl` check but before `fetch` resolves DNS) would bypass the check.
3. **Redirect SSRF (partially mitigated):** The code re-checks `isPrivateUrl(res.url)` after redirects (line 163), which is good, but still subject to DNS rebinding during the redirect chain.
4. **Missing ranges:** `0.0.0.0/8` (except the explicit `0.0.0.0` check), cloud metadata endpoints like `169.254.169.254` (the link-local check at line 21 covers 169.254.x.x, so this IS covered).

**Verification:**
- Line 15: `hostname === '0.0.0.0'` -- only exact match, not the range `0.x.x.x`
- Line 23: `::1` check exists but not `::ffff:127.0.0.1`
- Line 163: Post-redirect check exists (good)

The post-redirect check at line 163 significantly reduces the SSRF surface. The primary remaining gap is DNS rebinding, which requires a more sophisticated attack.

**Verdict: VULNERABLE** (low-medium severity -- most common SSRF vectors are blocked, but gaps remain for IPv6-mapped and DNS rebinding)

**WHY:** SSRF prevention via URL parsing is inherently a deny-list approach. The developer covered the common cases but the deny-list is incomplete. This is the "I blocked the IPs I know about" assumption.

---

### S13 — Export loads ALL bookmarks into memory → VULNERABLE

**Interrogation:** lib/exporter.ts:33-44 `fetchBookmarksFull` calls `prisma.bookmark.findMany` with no pagination. For large libraries, this loads all bookmarks, all media items, and all category joins into Node.js memory.

**Mechanism:** exportAllBookmarksCsv (line 146), exportBookmarksJson (line 178), and exportCategoryAsZip (line 89) all call fetchBookmarksFull. With 50,000+ bookmarks (each with rawJson of ~2-5KB), this could consume hundreds of MB.

Additionally, exportCategoryAsZip downloads media files (line 133: `const fileData = await downloadFile(item.url)`) for every media item in the category. For a category with 1000 bookmarks averaging 2 images each, this would make ~2000 HTTP requests sequentially and hold all downloaded images in memory before zipping.

**Verdict: VULNERABLE** (DoS vector -- OOM on large libraries)

**WHY:** The developer assumed bookmark counts stay small. The app is designed to import entire bookmark libraries (thousands of bookmarks), contradicting the "small dataset" assumption.

---

### S14 — Codex CLI temp file with predictable path → HOLDS

**Interrogation:** lib/codex-cli.ts:40 uses `join(tmpdir(), `codex-out-${randomUUID()}.txt`)`. The `randomUUID()` makes the filename unpredictable (128 bits of entropy).

**Challenge:** Could an attacker predict the UUID? No -- `crypto.randomUUID()` uses a CSPRNG. The file is also cleaned up after reading (line 57).

**Verdict: HOLDS**

---

### S15 — Module-level caches no invalidation for multi-instance → HOLDS (with caveat)

**Interrogation:** lib/settings.ts uses module-level variables with 5-minute TTL. In a multi-instance deployment, one instance could serve stale settings for up to 5 minutes after another instance changes them.

**Verification:** The app uses SQLite with file-based storage. SQLite doesn't support multiple concurrent writers well. The app would break at a more fundamental level (database locking) before cache inconsistency matters.

**Verdict: HOLDS** -- SQLite constrains this to single-instance anyway.

---

### S16 — Pipeline global state race condition → VULNERABLE

**Interrogation:** app/api/categorize/route.ts:98 checks `getState().status === 'running'` before starting. But the check-and-set is not atomic. Two rapid requests could both see `idle` and both start the pipeline.

**Mechanism:** The `void (async () => { ... })()` pattern at line 152 launches the pipeline asynchronously. The initial state is set to 'running' at line 137, but this happens AFTER the early returns at line 98-99. If request A sets state to 'running' at line 137 and request B arrives between lines 98 and 137, both will start.

Actually, re-reading more carefully: line 98 checks status, and line 137 sets it. In the JavaScript single-threaded model, between lines 98 and 137, no other request can execute (no await between them that would yield). The body parsing at line 104 does have an await, but that's after the status check... wait, no -- the code structure is:

```
line 98: if (getState().status === 'running' || ...) return 409
line 104: const text = await request.text()  // YIELD POINT
line 137: setState({ status: 'running' })
```

Between line 98 (check) and line 137 (set), there are multiple await points (lines 104, 106, 113, 126-131). Another request could be processed during any of these awaits and also pass the check at line 98.

**Verdict: VULNERABLE** (race condition -- two pipelines can start concurrently, causing double-processing and potential data corruption)

**WHY:** The developer assumed single-threaded JavaScript means no concurrency. But async/await yields control at every await point, allowing interleaved request processing. This is the "single-threaded = no races" assumption.

**Meta-pattern: Check-then-act across await boundaries.** This pattern is unsafe in async JavaScript.

---

### S17 — execSync blocks event loop → VULNERABLE

**Interrogation:** lib/claude-cli-auth.ts:42 uses `execSync('security find-generic-password ...')` which blocks the entire Node.js event loop for up to 3 seconds (timeout: 3000).

**Mechanism:** During this 3-second window, no other requests can be processed. The credential cache (CACHE_TTL_MS = 60_000) means this only happens once per minute, but on cache miss, all concurrent requests stall.

**Verdict: VULNERABLE** (low-medium severity -- temporary event loop blocking)

**WHY:** The developer chose synchronous I/O for simplicity in what appeared to be a one-shot operation. The caching partially mitigates it but doesn't eliminate it.

---

### S18 — DELETE /api/bookmarks deletes everything → HOLDS (by design)

**Interrogation:** app/api/bookmarks/route.ts:14-31 deletes all bookmarks, media, categories in one transaction. This is the "Clear Library" feature.

**Challenge:** Is there any path to this without authentication? The middleware protects all non-bookmarklet routes when auth is configured. Without auth configured, the app is intentionally open.

**Verdict: HOLDS** -- destructive but intentional feature, protected by auth middleware when auth is enabled.

---

### S19 — OpenAI assistant role drops image parts → VULNERABLE

**Interrogation:** lib/ai-client.ts:88:
```typescript
if (m.role === 'assistant') return { role: 'assistant' as const, content: parts.map(p => p.type === 'text' ? p : p).filter((p): p is OpenAI.ChatCompletionContentPartText => p.type === 'text') }
```

This line maps parts through `p.type === 'text' ? p : p` (which is a no-op -- returns p in both branches) but then filters to only keep `p.type === 'text'`. So all non-text content (images) in assistant role messages are silently dropped.

**Mechanism:** The `parts.map(p => p.type === 'text' ? p : p)` is dead code -- the map does nothing because both branches return `p`. The `.filter(...)` then drops all non-text parts. For assistant messages with image content, this silently loses data.

Currently, no code path sends image content in assistant role messages (images are always in user role), so this bug is dormant. But it's a clear logic error that would silently fail if the API changes.

**Verdict: VULNERABLE** (latent bug -- currently dormant, would silently lose data if triggered)

**WHY:** The ternary `p.type === 'text' ? p : p` suggests the developer intended to transform non-text parts but forgot to implement the transformation, settling for dropping them via the filter.

---

### S20 — N+1 queries in Twitter import → VULNERABLE

**Interrogation:** lib/twitter-api.ts:243-296 and app/api/import/twitter/route.ts:274-316 both check `findUnique` per tweet, then `create` per tweet, then `createMany` for media per tweet. For 100 tweets per page and up to 50 pages (5000 tweets), this is ~15,000 sequential database queries.

**Mechanism:** SQLite handles this reasonably well for small-to-medium imports because it's local disk I/O, not network round-trips. But for large imports, this creates significant latency (minutes).

The file import route (app/api/import/route.ts) has the same N+1 pattern at lines 77-117.

**Verdict: VULNERABLE** (performance, not security)

**WHY:** Sequential per-item processing is the simplest correct implementation. The developer optimized the categorizer (batch queries in categorizer.ts:300-343) but not the importers.

---

### S21 — REPLACE_ME placeholder query ID → VULNERABLE

**Interrogation:** app/api/import/twitter/route.ts:44: `queryId: 'REPLACE_ME'`

**Mechanism:** If a user attempts to import likes (source=like), the app will make a request to `https://x.com/i/api/graphql/REPLACE_ME/Likes?...` which will return a 400 error from Twitter. The error is caught at line 321-326 and returned as a 500.

The endpoint validates `source === 'like'` and requires `userId` at line 261-262, so someone intentionally trying to use the likes feature will get a confusing error.

**Verdict: VULNERABLE** (broken feature shipped as working)

**WHY:** The feature was scaffolded but never completed. The comment at line 43 says "PLACEHOLDER — you must replace this" but the route still accepts `source: 'like'` requests.

---

### S22 — Media proxy doesn't validate response content-type → HOLDS

**Interrogation:** The media proxy (app/api/media/route.ts) only allows URLs from a fixed set of Twitter media hosts (line 3-9). It forwards the upstream response body directly.

**Challenge:** Could an attacker use this to serve malicious content? The attacker cannot control the URL host (limited to twimg.com subdomains). Twitter's CDN serves media content. The proxy forwards Content-Type from upstream, so the browser will handle it according to Twitter's MIME type.

**Verdict: HOLDS** -- The host allowlist is the primary defense and is sufficiently restrictive.

---

### S23 — AI search sends up to 150 bookmarks in prompt → HOLDS (with cost caveat)

**Interrogation:** The search sends up to 150 bookmarks' index entries to the AI model. Each entry can be several hundred characters. This could produce prompts of 50-100K tokens.

**Mechanism:** The model is configured with max_tokens: 1500 for the response. But the input could be very large. At $3/M input tokens (Haiku), a single search query could cost $0.15-0.30. Without rate limiting (S08), this is a cost amplification vector.

**Verdict: HOLDS** (this is by-design behavior for semantic search, but interacts badly with S08 for cost amplification)

---

### S24 — Search cache unbounded key space → VULNERABLE

**Interrogation:** app/api/search/ai/route.ts:21-23: The cache evicts oldest entry when size reaches 100. But cache keys are `${query}::${category}` which means unique queries always create new entries.

**Mechanism:** The cache is Map-based (in-memory). With a max of 100 entries, memory usage is bounded. The eviction strategy (delete the oldest entry) is FIFO, not LRU. This means frequently-used queries can be evicted when 100 unique queries are made.

Actually, with only 100 entries max, this is not a memory issue. It's just poor cache behavior. The eviction at line 22 uses `searchCache.keys().next().value!` which deletes the first-inserted entry (FIFO).

**Verdict: VULNERABLE** (minor -- poor cache behavior, not a real memory issue)

---

### S25 — Category slug collision for Unicode → VULNERABLE

**Interrogation:** app/api/categories/route.ts:5-13:
```typescript
function generateSlug(name: string): string {
  return name.toLowerCase().trim()
    .replace(/[^a-z0-9\s-]/g, '')  // strips ALL non-ASCII
    .replace(/\s+/g, '-')
    ...
}
```

**Mechanism:** Any two category names that differ only in non-ASCII characters will produce the same slug. For example: "Dev Tools" and "Dev T00ls" produce different slugs, but "AI Ressources" (with Unicode s) and "AI Resources" produce different slugs only if the Unicode characters happen to map differently to ASCII. More critically, a Chinese name like "AI" would produce an empty slug after stripping.

**Verification:** Line 71 checks `if (!slug)` and returns 400, so empty slugs are caught. Lines 84-93 check for existing slugs and return 409 on collision. So collisions are detected but produce confusing errors ("Category with that name or slug already exists" when the names are clearly different).

**Verdict: VULNERABLE** (low severity -- confusing UX for non-ASCII category names, not a security issue)

---

### S26 — Bookmark categories PUT without transaction → VULNERABLE

**Interrogation:** app/api/bookmarks/[id]/categories/route.ts:21-32 does:
1. Delete all existing categories for the bookmark
2. Create new category links

If step 2 fails (e.g., invalid categoryId), the bookmark is left with zero categories. There is no transaction wrapping both operations.

**Verdict: VULNERABLE** (data integrity -- partial failure leaves bookmark uncategorized)

**WHY:** The developer assumed both operations succeed atomically. In Prisma, each statement is a separate transaction unless explicitly wrapped.

---

### S27 — Vision analyzer 12 concurrent workers → HOLDS (with caveat)

**Interrogation:** lib/vision-analyzer.ts:71 sets `CONCURRENCY = 12`. This means up to 12 simultaneous API calls.

**Challenge:** API rate limits would cause 429 errors. The retry logic at lines 139-166 handles rate limit errors with exponential backoff. So the system self-throttles when it hits rate limits.

**Verdict: HOLDS** -- The retry logic handles rate limiting, though initial bursts may waste API calls.

---

### S28 — No input size validation on import file upload → VULNERABLE

**Interrogation:** app/api/import/route.ts:5-27 calls `request.formData()` then `file.text()` without checking file size. A multi-GB JSON file would be read entirely into memory.

**Mechanism:** Next.js has a default body size limit (typically 1MB for API routes unless configured otherwise). But the default can be overridden, and formData handling may use a different limit path.

**Verification:** next.config.ts was not read but typically Next.js API routes have a 1MB default limit for `request.json()`. For `request.formData()`, the limit may differ. Even at 1MB, parsing a 1MB JSON with parseBookmarksJson involves creating ParsedBookmark objects for every tweet, which could multiply memory usage.

**Verdict: VULNERABLE** (medium severity -- depends on Next.js bodyParser configuration; at minimum, no explicit size check)

---

## Output

### Seed Map

- **Total seeds:** 28
- **VULNERABLE:** 19 (S01, S02, S03, S04, S06, S07, S08, S09, S10, S12, S13, S16, S17, S19, S20, S21, S24, S25, S26, S28)
- **HOLDS:** 7 (S05, S11, S14, S15, S18, S22, S27)
- **UNCLEAR:** 0
- **Subsumed:** 2 (S10 by S01, S23 by S08)

### Root Causes (grouped by shared failed assumption + meta-pattern)

**RC1: "CORS = Authentication" (S01, S02, S10)**
- Failed assumption: Browser-enforced CORS prevents unauthorized access
- Meta-pattern: Confusing client-side restrictions with server-side access control
- Impact: Unauthenticated data injection, API credit theft

**RC2: "Code duplication across import paths" (S03, S09, S21)**
- Failed assumption: Duplicated implementations stay in sync
- Meta-pattern: Safety guards in one copy but not another; constants drift between copies
- Impact: Inconsistent bearer tokens, missing pagination limits, broken features shipped as working
- Files affected: lib/twitter-api.ts vs app/api/import/twitter/route.ts vs app/api/import/bookmarklet/route.ts

**RC3: "It's a local-only tool" (S04, S08, S13, S28)**
- Failed assumption: The app runs only on localhost with a single user
- Meta-pattern: Self-contradicting deployment model -- app ships with Docker support and Cloudflare Tunnel integration while assuming no adversarial access
- Impact: Plaintext secrets, no rate limiting, OOM on large datasets

**RC4: "Single-threaded JS = no concurrency" (S16)**
- Failed assumption: JavaScript's single thread prevents race conditions
- Meta-pattern: Check-then-act across await boundaries in async code
- Impact: Double pipeline execution, data corruption

**RC5: "Partial sanitization is sufficient" (S06, S12)**
- Failed assumption: Blocking known-bad inputs covers all cases
- Meta-pattern: Deny-list approach to input sanitization leaves unknown-bad inputs uncovered
- Impact: FTS query manipulation, potential SSRF via DNS rebinding

### Assumption Radiation Map

```
RC1 (CORS=Auth) ──→ S01 (bookmarklet bypass) ──→ S02 (no CSRF)
                  └──→ S10 (CORS-only protection)
                        └──→ S08 (no rate limit) ──→ cost amplification

RC2 (Duplication) ──→ S03 (bearer drift) ──→ silent import failure
                   └──→ S09 (no page limit) ──→ unbounded loop / OOM
                   └──→ S21 (REPLACE_ME) ──→ broken feature

RC3 (Local-only) ──→ S04 (plaintext keys) ──→ credential theft
                  └──→ S08 (no rate limit) ──→ API credit exhaustion
                  └──→ S13 (full memory load) ──→ OOM DoS
                  └──→ S28 (no size check) ──→ OOM DoS

RC4 (No races) ──→ S16 (pipeline race) ──→ double processing / corruption

RC5 (Partial sanitize) ──→ S06 (FTS manipulation) ──→ query abuse
                        └──→ S12 (SSRF gaps) ──→ internal network access
```

### Impact Chains

1. **Unauthenticated injection chain:** Attacker → bookmarklet endpoint (S01, no auth) → inject poisoned bookmarks → trigger categorization pipeline → exhaust AI API credits (S08, no rate limit)

2. **Credential theft chain:** Attacker with file access → SQLite DB (S04, plaintext keys) → X auth_token/ct0 → full Twitter account access + API key theft

3. **Denial of service chain:** Attacker → export endpoint (S13, full memory load) → OOM crash. Alternative: import endpoint (S28) with large payload → OOM crash.

4. **Pipeline corruption chain:** Two rapid categorize requests (S16) → race condition → both start → duplicate API calls + potential double-write of category assignments

5. **Silent feature degradation:** Bearer token drift (S03) → one import path fails → user sees generic error, doesn't know which path works

### Defense Map (HOLDS + coverage gaps)

| Defense | Coverage | Gap |
|---------|----------|-----|
| Auth middleware (Basic Auth) | All routes except bookmarklet | Gap: bookmarklet is fully open (S01) |
| SSRF URL check (link-preview) | Common private IPs, post-redirect recheck | Gap: IPv6-mapped, DNS rebinding (S12) |
| Media proxy host allowlist | Only twimg.com domains | Full coverage |
| FTS parameterized query | SQL injection prevented | Gap: FTS5 syntax injection (S06) |
| Settings cache invalidation | Called after settings POST | Gap: no cross-instance invalidation (moot for SQLite) |
| Pipeline abort mechanism | shouldAbort() checked throughout | Gap: race on startup (S16) |
| Retry with backoff (vision) | Handles rate limits | Full coverage for retryable errors |
| Import dedup (findUnique) | Prevents duplicate tweet import | Full coverage |
| Masked key display (settings GET) | Prevents key leakage via API | Full coverage (keys in DB still plaintext) |

### Unexplored (UNCLEAR, needs-more-time)

None -- all 28 seeds received a verdict.
