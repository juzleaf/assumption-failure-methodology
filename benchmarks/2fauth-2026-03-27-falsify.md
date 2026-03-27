# AFM Falsify Report: 2FAuth Worker

> Target: `/tmp/afm-benchmark-round3/2fauth-v31/`
> Date: 2026-03-27
> Method: AFM Falsify (Seed + Interrogate + Falsify)
> Blind test -- no prior context about what to look for.

---

## Phase 1: Seed List

| # | Seed | File:Line |
|---|------|-----------|
| S01 | CORS origin reflector: `origin: (origin) => origin` reflects any origin | `backend/src/app/index.ts:34` |
| S02 | CSRF Double Submit Cookie uses `SameSite: Lax` + non-httpOnly csrf_token | `backend/src/features/auth/authRoutes.ts:96-101` |
| S03 | Rate limiter silently skips when DB is null or on DB error | `backend/src/shared/middleware/rateLimitMiddleware.ts:17-21,104-109` |
| S04 | Rate limiter key uses `CF-Connecting-IP` header (spoofable without Cloudflare) | `backend/src/shared/middleware/rateLimitMiddleware.ts:24` |
| S05 | `resetRateLimit` uses wrong path prefix (`/api/auth/` vs actual `/api/oauth/`) | `backend/src/features/auth/authRoutes.ts:106` |
| S06 | OAUTH_ALLOW_ALL value `'2'` bypasses whitelist but not health check | `backend/src/features/auth/authService.ts:91` vs `backend/src/shared/utils/health.ts:60` |
| S07 | `encryptionKey` exposed to frontend via `/api/oauth/me` when `!isEmergencyConfirmed` | `backend/src/features/auth/authRoutes.ts:143-149` |
| S08 | `encryptionKey` exposed in OAuth callback response when `needsEmergency` | `backend/src/features/auth/authService.ts:79` |
| S09 | Telegram login allows 24-hour auth_date window (86400s) | `backend/src/features/auth/providers/telegramProvider.ts:78` |
| S10 | Telegram whitelist field is `['id']` -- but whitelist checks `email` or `username`, not `id` | `backend/src/features/auth/providers/telegramProvider.ts:9` vs `backend/src/features/auth/authService.ts:99-109` |
| S11 | Logout endpoint has no CSRF/auth protection | `backend/src/features/auth/authRoutes.ts:118` |
| S12 | Backup OAuth callbacks use `postMessage(msg, '*')` with wildcard origin | `backend/src/features/backup/backupRoutes.ts:29,147` |
| S13 | Backup OAuth callback HTML injects error_description into inline JS via template literal | `backend/src/features/backup/backupRoutes.ts:106-108` |
| S14 | `vaultService.exportAccounts` overrides `SECURITY_CONFIG.MIN_EXPORT_PASSWORD_LENGTH` to 5 (config says 12) | `backend/src/features/vault/vaultService.ts:188` |
| S15 | SQL injection vector in search: `like(vault.service, '%${search}%')` | `backend/src/shared/db/repositories/vaultRepository.ts:39-41` |
| S16 | Import type `'raw'` handled AFTER try-catch, gets raw JSON.parse without validation | `backend/src/features/vault/vaultService.ts:341` |
| S17 | WebAuthn passkey login skips whitelist verification | `backend/src/features/auth/webAuthnService.ts:143-152` |
| S18 | WebAuthn credential deletion/update not scoped to current user | `backend/src/features/auth/webAuthnService.ts:216-230` |
| S19 | `executeManualBackup` accepts arbitrary `filename` and `content` from client without validation | `backend/src/features/backup/backupService.ts:305-307` |
| S20 | Telegram webhook has no secret_token verification (commented out) | `backend/src/features/telegram/telegramRoutes.ts:11-13` |
| S21 | Health check endpoint exposes detailed security issues to unauthenticated users | `backend/src/features/health/healthRoutes.ts:9-21` |
| S22 | Backup provider config encryption fallback to JWT_SECRET: `this.env.ENCRYPTION_KEY || this.env.JWT_SECRET` | `backend/src/features/backup/backupService.ts:122,196,216,243,278,303,354,374,399,418` |
| S23 | `codeVerifier` returned to frontend in authorize response JSON | `backend/src/features/auth/authRoutes.ts:55` |
| S24 | Vault operations have no per-user scoping -- all authenticated users share one vault | Design assumption |
| S25 | `add-from-uri` endpoint has relaxed secret validation vs `parseOTPAuthURI` strict check | `backend/src/features/vault/vaultRoutes.ts:129-149` |

---

## Phase 2: Interrogate + Falsify

---

### S01: CORS Origin Reflection

**Interrogate:**
- *Why?* CORS with `credentials: true` requires a non-wildcard Origin. The dev chose to reflect any incoming Origin.
- *What does it do?* Any website can make credentialed cross-origin requests to this API.
- *What does it promise?* That Cookie-based auth is safe from cross-site request forgery.

**Assumption:** "CSRF is handled by the Double Submit Cookie pattern, so reflected origin is safe."

**FORWARD:** If a malicious site issues `fetch('https://victim.2fauth.app/api/vault/', {credentials:'include'})`, the browser sends the auth cookie AND the csrf_token cookie. The site also needs to send the X-CSRF-Token header. Can it read the csrf_token cookie? The csrf_token cookie is `SameSite: Lax`, `httpOnly: false`, `secure: true`, set for path `/`. With reflected CORS + credentials, the browser sends the cookie, but the attacker's JS on evil.com cannot read a cross-site cookie from the response (Same-Origin Policy on cookies prevents `document.cookie` access across origins). However, the CORS response includes `Access-Control-Allow-Credentials: true` and the reflected origin, so the browser WILL include cookies in the request. The attacker's JS cannot read `document.cookie` for the victim's domain, but can the attacker somehow obtain the CSRF token?

The CSRF cookie is sent to the server with the request, but the attacker needs to also provide it in the X-CSRF-Token header. Without reading the victim domain's cookie, the attacker cannot set the header. This holds as long as the attacker cannot exfiltrate the CSRF value.

**FALSIFY (attack HOLDS):**
- Can the attacker read the CSRF token via a CORS response? The `/api/oauth/me` endpoint does not return the CSRF token in the response body. However, any response from the API will have the `Set-Cookie: csrf_token=...` header. With CORS + credentials, can JS read the Set-Cookie header? No -- `Set-Cookie` is a forbidden response header name; `fetch()` cannot access it even with CORS.
- Can the CSRF be read via other means? The CSRF is a cookie value, readable only by same-origin scripts. Cross-origin scripts cannot access another domain's cookies.
- Alternative: If there's an XSS on the victim site, the attacker reads `document.cookie` since `csrf_token` is not httpOnly. But that requires XSS first (separate issue).

**Verdict: HOLDS -> HARDENED**

The CORS reflection is defended by the Double Submit Cookie pattern. An attacker on a different origin cannot read the csrf_token cookie to set the X-CSRF-Token header. The defense survived: (1) cross-origin fetch cannot read Set-Cookie, (2) document.cookie is same-origin-only, (3) no CSRF token is leaked in response bodies.

Note: This becomes VULNERABLE immediately if XSS exists (S12/S13 are XSS vectors). Conditional dependency documented.

---

### S02: CSRF Double Submit Cookie with SameSite Lax

**Interrogate:**
- *Why?* Standard pattern for SPAs with Cookie-based auth.
- *What does it do?* Every mutating request must include X-CSRF-Token header matching the csrf_token cookie.
- *What does it promise?* Cross-origin attackers cannot forge authenticated requests.

The CSRF check is in `authMiddleware` (line 14-18 of auth.ts), applied to all protected routes. Unprotected routes (health, providers list, OAuth callbacks) don't need it.

**FALSIFY:**
Checked all routes: `vault/*`, `backups/*` (after `authMiddleware` middleware), `emergency/*`, `tools/*` -- all use `authMiddleware`. The backup OAuth GET callbacks bypass auth/CSRF, but they verify state cookies independently.

The logout endpoint (`auth.post('/logout')`) does NOT use `authMiddleware`:
```
auth.post('/logout', (c) => { ... })
```
This means logout is not CSRF-protected. An attacker can forge a logout request (SameSite: Lax allows POST from top-level navigations on some browsers, but more importantly, the auth_token cookie IS sent). However, logout is a low-impact target.

**Verdict: HOLDS -> HARDENED** (core CSRF pattern is sound; logout without CSRF is documented in S11 separately)

---

### S03: Rate Limiter Silent Skip on DB Error

**Interrogate:**
- *Why?* Availability over security -- don't break the app if rate limit DB isn't configured.
- *What does it do?* If `db` is null or any DB error occurs (except 429), the request proceeds without rate limiting.
- *What does it promise?* Brute force protection on login endpoints.

**FALSIFY:**
If the D1 database is misconfigured, has a transient error, or if `rate_limits` table doesn't exist, ALL rate limiting silently disappears. The catch block at line 104-109:
```typescript
catch (e: any) {
    if (e instanceof AppError && e.statusCode === 429) throw e;
    console.error('[RateLimit] Database error:', e.message);
}
await next(); // proceeds regardless
```

Scenario: D1 database has a transient connectivity issue (common in serverless). Every login attempt during the outage bypasses rate limiting entirely. An attacker who can trigger or wait for DB errors gets unlimited brute force attempts.

However, this system uses OAuth-only login (no password brute force target). The rate limit protects against OAuth callback spam, which has limited attacker value since the attacker needs valid OAuth codes.

**Verdict: SUSPECTED**

The silent skip is a real design flaw, but impact is limited because the system is OAuth-only with no direct password to brute force. Cannot construct a concrete exploit with measurable impact beyond theoretical resource exhaustion.

---

### S04: CF-Connecting-IP Spoofing

**Interrogate:**
- *Why?* Cloudflare Workers receive the true client IP via this header.
- *What does it do?* Rate limit key is built from `CF-Connecting-IP`.
- *What does it promise?* Per-IP rate limiting.

**FALSIFY:**
On Cloudflare Workers, `CF-Connecting-IP` is set by Cloudflare's edge and CANNOT be spoofed by the client. The header is stripped and replaced at the edge.

In Docker mode (server.ts), requests arrive via Node.js directly. There is no Cloudflare edge. The client can send a fake `CF-Connecting-IP` header, and the rate limiter will use it as the key. This means an attacker can bypass rate limits by rotating the header value.

Concrete exploit (Docker deployment):
```
curl -X POST https://docker-host/api/oauth/callback/github \
  -H "CF-Connecting-IP: fake-ip-1" \
  -H "Content-Type: application/json" \
  -d '{"code":"x","state":"y"}'
# Repeat with fake-ip-2, fake-ip-3, etc. -- unlimited attempts
```

**Verdict: PROVEN**

**Proof:** In Docker mode (server.ts), no reverse proxy strips/overrides `CF-Connecting-IP`. The rate limiter at `rateLimitMiddleware.ts:24` reads `c.req.header('CF-Connecting-IP') || 'unknown'` directly from the request. An attacker sends arbitrary values in this header, generating a unique rate-limit key per request, completely bypassing rate limiting.

**Root cause WHY:** The developers assumed the application would always run behind Cloudflare's edge network, which strips client-sent `CF-Connecting-IP`. The Docker deployment path breaks this assumption.

**Meta-pattern:** "Assumes deployment environment guarantees" -- infrastructure security assumptions that hold in one deployment mode but fail in another.

---

### S05: Rate Limit Reset Key Mismatch

**Interrogate:**
- *Why?* After successful login, rate limit counters should be cleared.
- *What does it do?* Calls `resetRateLimit(c, 'rl:${clientIp}:/api/auth/callback/${providerName}')`.
- *What does it promise?* Successful login clears the lockout counter.

**FALSIFY:**
The OAuth callback route is mounted at: `app.route('/api/oauth', authRoutes)` (index.ts:91), and the route handler is `auth.post('/callback/:provider', ...)` (authRoutes.ts:60). So the actual request path is `/api/oauth/callback/:provider`.

But the reset key is: `rl:${clientIp}:/api/auth/callback/${providerName}` (authRoutes.ts:106).

The rate limit key is: `rl:${clientIp}:${path}` where path = `/api/oauth/callback/:provider` (rateLimitMiddleware.ts:26).

**The reset key uses `/api/auth/` but the actual path is `/api/oauth/`. These are different strings. The reset never matches the rate limit key. The counter is never cleared after successful login.**

Similarly for WebAuthn at line 225: `rl:${clientIp}:/api/auth/webauthn/login/verify` vs actual path `/api/oauth/webauthn/login/verify`.

Concrete scenario:
1. User tries to login 4 times (max is 5).
2. On the 4th try, login succeeds.
3. `resetRateLimit` tries to delete key `rl:1.2.3.4:/api/auth/callback/github`.
4. The actual stored key is `rl:1.2.3.4:/api/oauth/callback/github`.
5. Reset does nothing. The counter stays at 4.
6. Next login attempt (5th overall) triggers lockout even though the 4th was successful.

**Verdict: PROVEN**

**Proof:** Path prefix mismatch: `/api/auth/` in reset key vs `/api/oauth/` in actual route. The `resetRateLimit` function deletes a key that was never inserted. After a successful login, the rate limit counter remains, and subsequent legitimate login attempts will be rate-limited despite prior success.

- Rate limit key builder: `rateLimitMiddleware.ts:26` -- `rl:${clientIp}:${path}` uses the actual request path.
- Route mount: `index.ts:91` -- `app.route('/api/oauth', authRoutes)`.
- Reset call: `authRoutes.ts:106` -- hardcoded `/api/auth/callback/`.

**Root cause WHY:** The route prefix was renamed from `/api/auth` to `/api/oauth` at some point, but the hardcoded reset key strings were not updated.

**Meta-pattern:** "Hardcoded path strings diverge from actual routing" -- when path prefixes are refactored, all string references must be updated.

---

### S06: OAUTH_ALLOW_ALL value '2' Bypass

**Interrogate:**
- *Why?* Multiple truthy values are accepted for configuration flexibility.
- *What does it do?* `authService.ts:91` accepts `'true'`, `'1'`, and `'2'` as bypass values. Health check at `health.ts:60` only checks `'true'` and `'1'`.
- *What does it promise?* Health check catches dangerous open-registration configurations.

**FALSIFY:**
If an admin sets `OAUTH_ALLOW_ALL=2`:
1. Health check (`health.ts:60`): `String(env.OAUTH_ALLOW_ALL).toLowerCase()` is `'2'`. `'2' === 'true'` is false, `'2' === '1'` is false. Health check passes -- no warning.
2. Whitelist check (`authService.ts:91`): `allowAllStr === '2'` is true. Whitelist is bypassed. Any OAuth user can login.

This means `OAUTH_ALLOW_ALL=2` creates an open-registration system with no health check warning.

**Verdict: PROVEN**

**Proof:** Setting `OAUTH_ALLOW_ALL=2` in environment variables:
- `authService.ts:91`: `if (allowAllStr === 'true' || allowAllStr === '1' || allowAllStr === '2')` -- evaluates true, whitelist skipped.
- `health.ts:60`: `if (String(env.OAUTH_ALLOW_ALL).toLowerCase() === 'true' || env.OAUTH_ALLOW_ALL === '1')` -- evaluates false, no warning issued.

Result: Any user who completes OAuth flow can log in and access ALL vault secrets. The health check gives a clean bill, showing no security issues.

**Root cause WHY:** The `'2'` value was added to authService as possibly a "demo mode" indicator but was never added to the corresponding health check.

**Meta-pattern:** "Security checks with multiple code paths diverge" -- when the same configuration is checked in multiple places with different logic, gaps appear.

---

### S07 + S08: ENCRYPTION_KEY Exposure to Frontend

**Interrogate:**
- *Why?* During initial setup ("emergency" flow), the user must record their encryption key for disaster recovery.
- *What does it do?* Returns `ENCRYPTION_KEY` in the JSON response of login callback and `/api/oauth/me`.
- *What does it promise?* The key is only exposed temporarily during first-time setup.

**FALSIFY:**
The exposure path for `/api/oauth/me` (authRoutes.ts:143):
```typescript
const encryptionKey = !isEmergencyConfirmed ? c.env.ENCRYPTION_KEY : undefined;
```

This means EVERY authenticated request to `/me` returns the full encryption key as long as the emergency flow hasn't been confirmed. If the user delays confirming emergency setup, the key is exposed on every page load.

Furthermore, in the login callback response (authService.ts:79):
```typescript
...(needsEmergency && { encryptionKey: this.env.ENCRYPTION_KEY })
```

The ENCRYPTION_KEY is the master key that encrypts ALL vault secrets in the database. Exposing it in API responses means:
1. It appears in browser DevTools network tab.
2. It may be logged by proxies, CDNs, or monitoring tools.
3. Any XSS vulnerability immediately yields the master encryption key.
4. Browser extensions with network access can capture it.

**Verdict: PROVEN**

**Proof:** The `ENCRYPTION_KEY` environment variable (the master key protecting all 2FA secrets at rest) is sent verbatim in the JSON response body of:
- `POST /api/oauth/callback/:provider` -- `authService.ts:79`
- `GET /api/oauth/me` -- `authRoutes.ts:143`
- `POST /api/oauth/webauthn/login/verify` -- `webAuthnService.ts:173`

Condition: `!isEmergencyConfirmed` (first-time setup not completed).

Impact: The key is visible in browser network inspector, potentially logged by intermediate infrastructure, and capturable by any XSS or browser extension. This is the key that encrypts/decrypts ALL TOTP secrets stored in the database.

**Root cause WHY:** The developers needed a way to show users their encryption key during initial setup. Rather than a one-time display mechanism, they embedded it in standard API responses that are called repeatedly.

**Meta-pattern:** "Secrets transmitted in regular API responses" -- master keys should never traverse network responses; use a dedicated, auditable, one-time-only display channel.

---

### S09: Telegram 24-Hour Auth Window

**Interrogate:**
- *Why?* Telegram docs say `auth_date` should be checked. 24 hours gives flexibility.
- *What does it do?* Accepts Telegram login data signed up to 24 hours ago.
- *What does it promise?* Timely authentication.

**FALSIFY:**
Telegram's official recommendation is that the auth_date should be checked to be "not too old" (their example uses 1 day). A 24-hour window means a stolen/leaked Telegram login hash is replayable for a full day. However, the Telegram hash is cryptographically signed with the bot token, so it can't be forged. The risk is replay of a legitimately obtained hash.

Impact is limited: an attacker would need to intercept the exact hash data, and the OAuth state cookie provides additional CSRF protection on the callback.

**Verdict: SUSPECTED**

Replay window is larger than necessary (recommended: 5-15 minutes), but constructing a concrete exploit requires intercepting the Telegram auth hash before the user completes login, which is a niche scenario.

---

### S10: Telegram Users Cannot Be Whitelisted

**Interrogate:**
- *Why?* Telegram doesn't provide email. The provider declares `whitelistFields = ['id']`.
- *What does it do?* The whitelist checker only looks at `email` and `username` fields (authService.ts:104-109). It never checks `id`.
- *What does it promise?* Telegram users can be whitelisted.

**FALSIFY:**
Code path in `authService.ts:88-119`:
```typescript
private verifyWhitelist(userInfo: OAuthUserInfo, whitelistFields: string[]) {
    // ...
    if (whitelistFields.includes('email') && userEmail && allowedIdentities.includes(userEmail)) {
        isAllowed = true;
    }
    if (whitelistFields.includes('username') && userName && allowedIdentities.includes(userName)) {
        isAllowed = true;
    }
    if (!isAllowed) {
        throw new AppError('unauthorized_user', 403);
    }
```

Telegram provider: `whitelistFields = ['id']`. The whitelist only checks `email` and `username`. There is no `if (whitelistFields.includes('id') ...)` branch.

TelegramProvider returns: `email: ''` and `username: params.get('username') || 'tg_user_${id}'`.

Scenario 1: Admin adds Telegram user's numeric ID (e.g., `123456`) to OAUTH_ALLOWED_USERS.
- `whitelistFields.includes('email')` with field `'id'` -- false.
- `whitelistFields.includes('username')` with field `'id'` -- false.
- `isAllowed` stays false. User is rejected.

Scenario 2: Admin adds Telegram username (e.g., `myuser`) to OAUTH_ALLOWED_USERS.
- `whitelistFields.includes('username')` with field `'id'` -- false (whitelistFields is `['id']`, not `['username']`).
- User is still rejected!

**This means Telegram login ALWAYS fails with `unauthorized_user` unless OAUTH_ALLOW_ALL is enabled.** Telegram login is effectively broken for any production deployment using whitelists.

**Verdict: PROVEN**

**Proof:** `TelegramProvider.whitelistFields = ['id']` (telegramProvider.ts:9). `verifyWhitelist` only checks fields `'email'` and `'username'` (authService.ts:104-109). Since `whitelistFields` contains only `'id'`, neither branch is entered, and `isAllowed` remains `false`. Every Telegram login throws `unauthorized_user: 403`.

The only way Telegram login works is with `OAUTH_ALLOW_ALL=true|1|2`, which disables ALL access control.

**Root cause WHY:** The whitelist system was designed for OAuth providers that return email/username. Telegram was added later with `id` as its identifier, but the whitelist checker was never extended to handle `id`.

**Meta-pattern:** "Extension point not wired to all dimensions" -- when a new variant is added to a polymorphic system, all consumers of the variant's interface must be updated.

---

### S11: Logout Without CSRF/Auth

**Interrogate:**
- *Why?* Allow users to log out even if CSRF token is stale.
- *What does it do?* Deletes auth_token and csrf_token cookies.
- *What does it promise?* Only the authenticated user triggers their own logout.

**FALSIFY:**
A CSRF attack can cause a victim to visit `https://victim-app/api/oauth/logout` via a form POST. The SameSite: Lax cookie is sent for top-level navigation POSTs. The server deletes the auth cookies.

Impact: Denial of service -- attacker can force-logout any user who clicks a malicious link. This is annoying but not a security breach (no data is exposed or modified).

**Verdict: SUSPECTED**

Logout CSRF is a well-known low-severity issue. No data exposure, only user inconvenience. Can construct the attack but impact doesn't warrant PROVEN severity for a security audit.

---

### S12: postMessage with Wildcard Origin

**Interrogate:**
- *Why?* The OAuth callback popup needs to send data back to the opener window.
- *What does it do?* Sends `window.opener.postMessage(message, '*')` with the refresh token.
- *What does it promise?* Only the parent application receives the token.

**FALSIFY:**
The `*` target origin means ANY window that happens to be the opener receives the message. If an attacker can control the opener reference (e.g., by opening the OAuth flow in a controlled context), they receive the refresh token.

Attack chain:
1. Attacker opens `https://victim-app/api/backups/oauth/google/callback?code=VALID&state=VALID` from their site.
2. But wait -- the state must match the cookie. The attacker doesn't have the state cookie.
3. The state validation would fail first, preventing token exchange.

The `*` origin is still a defense-in-depth violation, but the state cookie check prevents the actual exploit.

However, there's also `BroadcastChannel` being used. BroadcastChannel messages are received by ALL same-origin pages. If the attacker has injected any script on the same origin (XSS), they can listen on the channel and steal the refresh token.

**Verdict: SUSPECTED**

The wildcard postMessage is a defense-in-depth violation, but the state cookie check prevents the primary attack. Concrete exploitation requires XSS on the same origin first.

---

### S13: Template Literal XSS in Backup OAuth Error

**Interrogate:**
- *Why?* Error messages from token exchange are displayed to the user.
- *What does it do?* Injects `errData.error_description` into an inline `<script>` via template literal.
- *What does it promise?* Only safe text is displayed.

**FALSIFY:**
At `backupRoutes.ts:106-108`:
```typescript
const msg = { type: 'GDRIVE_AUTH_ERROR', message: 'Token exchange failed: ${errData.error_description || errData.error}' };
```

This is a **server-side** template literal (TypeScript backtick string), not a client-side one. The `errData` comes from Google's token endpoint response. If Google's error response contains JavaScript-breaking characters (like `'`, `</script>`, etc.), they would be injected into the inline script.

However, Google's OAuth error responses are controlled by Google and contain standard error strings like `invalid_grant`. The attacker controls the `code` parameter but not Google's error response format.

Wait -- actually, this is the SERVER rendering HTML with the template literal. The variable `errData.error_description` is interpolated into a JavaScript string inside `<script>` tags at HTML generation time. If `errData.error_description` contains `');alert(1)//` or `</script><script>alert(1)//`, it would break out of the JS context.

But `errData` comes from Google's API response, which the attacker doesn't control. The attacker only controls the `code` parameter sent to Google, and Google returns its own error format.

Similar patterns exist for Microsoft (line 266), Baidu (line 413), and Dropbox (line 557).

**Verdict: SUSPECTED**

The code pattern IS an XSS vulnerability (unsanitized data in inline script), but the data source (OAuth provider error responses) is not attacker-controlled under normal circumstances. If a provider returns unexpected content (or the fetch is MITM'd), XSS would execute.

---

### S14: Export Password Minimum Length Override

**Interrogate:**
- *Why?* Unclear -- the local override shadows the global config.
- *What does it do?* `vaultService.ts:188` declares `const SECURITY_CONFIG = { MIN_EXPORT_PASSWORD_LENGTH: 5 };`, shadowing the global config's value of 12.
- *What does it promise?* Exported files are protected by sufficiently strong passwords.

**FALSIFY:**
The global `SECURITY_CONFIG.MIN_EXPORT_PASSWORD_LENGTH` is 12 (config.ts:8). But `exportAccounts` creates a local variable with the same name set to 5.

```typescript
async exportAccounts(type: string, password?: string) {
    const SECURITY_CONFIG = { MIN_EXPORT_PASSWORD_LENGTH: 5 }; // shadows global!
    if (type === 'encrypted') {
        if (!password || password.length < SECURITY_CONFIG.MIN_EXPORT_PASSWORD_LENGTH) {
```

A user can export their entire vault encrypted with a 5-character password. Given PBKDF2 with 100k iterations, a 5-char password is brute-forceable.

This also creates inconsistency: the backup provider requires 6-char minimum (`backupService.ts:221`), but direct export allows 5-char.

**Verdict: PROVEN**

**Proof:** At `vaultService.ts:188`, a local `SECURITY_CONFIG` object shadows the imported global config, reducing `MIN_EXPORT_PASSWORD_LENGTH` from 12 to 5. A user exporting vault data with type `'encrypted'` can use a 5-character password. This exported file contains ALL decrypted 2FA secrets.

Input: `POST /api/vault/export` with body `{"type":"encrypted","password":"12345"}` -- accepted, returns encrypted vault with all secrets.

**Root cause WHY:** Likely a development shortcut that was never reverted. The local variable was probably created for testing with short passwords.

**Meta-pattern:** "Local variable shadows security configuration" -- when security constants are redefined locally, the global policy is silently bypassed.

---

### S15: SQL Injection via Search Parameter

**Interrogate:**
- *Why?* The search query uses Drizzle ORM's `like()` function.
- *What does it do?* Constructs `LIKE '%${search}%'`.
- *What does it promise?* Safe query construction.

**FALSIFY:**
Drizzle ORM's `like()` function parameterizes the value. The template-literal-looking syntax `%${search}%` is NOT string interpolation into raw SQL -- it's a JavaScript string that becomes a parameterized value in the underlying SQL.

The generated SQL would be: `WHERE service LIKE ?` with parameter `%searchterm%`. SQLite's LIKE wildcards (`%`, `_`) in the search term can affect query behavior (e.g., searching for `%` returns everything), but this is not SQL injection -- it's LIKE pattern injection, which has minimal security impact (the user is already authenticated and can see all their data).

**Verdict: HOLDS -> HARDENED**

Drizzle ORM parameterizes the LIKE value. No SQL injection is possible. LIKE wildcard injection (`%`, `_`) is cosmetic only since authenticated users already have full vault access.

---

### S16: Raw Import Type After Try-Catch

**Interrogate:**
- *Why?* The `'raw'` type appears to be a development/internal import mechanism.
- *What does it do?* `vaultService.ts:341`: `if (type === 'raw') validAccounts = JSON.parse(content);` -- executed AFTER the try-catch block.
- *What does it promise?* Import data is validated before insertion.

**FALSIFY:**
When `type === 'raw'`:
1. The try-catch block at line 252 is entered. Since `type` doesn't match any branch inside, `validAccounts` stays `[]`.
2. At line 341, OUTSIDE the try-catch: `validAccounts = JSON.parse(content)` runs WITHOUT any try-catch protection.
3. If `content` is not valid JSON, an unhandled error propagates.
4. If `content` is valid JSON, the raw parsed data goes into the validation loop at line 350, which checks `validateBase32Secret(acc.secret)`.

The validation at line 350-363 still runs, so arbitrary data can't fully bypass validation. But the JSON.parse outside try-catch means malformed input causes an unhandled error (500 instead of 400).

More importantly, the `'raw'` type is not listed in the frontend import options, suggesting it's a backdoor/debug endpoint. There's no auth bypass -- vault routes require `authMiddleware`.

**Verdict: SUSPECTED**

The `raw` type is an undocumented import pathway that bypasses the try-catch error handling. Invalid JSON causes 500 errors instead of graceful 400s. However, validation still runs on the parsed data, and auth is required.

---

### S17: WebAuthn Passkey Login Skips Whitelist

**Interrogate:**
- *Why?* The WebAuthn login path was added separately from OAuth login.
- *What does it do?* After WebAuthn verification succeeds, it directly issues a JWT without calling `verifyWhitelist`.
- *What does it promise?* Only whitelisted users can access the system.

**FALSIFY:**
In `webAuthnService.ts:134-174`, after successful authentication:
```typescript
if (verification.verified && verification.authenticationInfo) {
    // ... update counter
    const userEmail = credential.userId;
    const token = await this.generateSystemToken({...});
    // NO whitelist check
    return { success: true, token, ... };
}
```

Compare with OAuth flow in `authService.ts:61`:
```typescript
this.verifyWhitelist(userInfo, provider.whitelistFields);
```

The whitelist check is completely absent from the WebAuthn login path.

Attack scenario:
1. Admin configures `OAUTH_ALLOWED_USERS=admin@example.com`.
2. User `attacker@example.com` logs in via OAuth (e.g., when OAUTH_ALLOW_ALL was briefly enabled, or via Passkey registration while already logged in).
3. Admin removes `attacker@example.com` from whitelist and sets OAUTH_ALLOW_ALL=false.
4. Attacker can still log in via their registered Passkey because WebAuthn login doesn't check the whitelist.

Furthermore, Passkey registration requires being logged in (`authMiddleware`), but there's no check that the registering user's email matches the authenticated user. And Passkey login has no whitelist check at all.

**Verdict: PROVEN**

**Proof:** `webAuthnService.ts:134-174` -- the `verifyAuthenticationResponse` method issues a JWT at line 147-152 without any call to `verifyWhitelist`. The function `verifyWhitelist` exists only in `authService.ts:88` and is only called from `handleOAuthCallback` (line 61).

Code path: WebAuthn login verify -> DB lookup passkey -> cryptographic verification -> issue JWT. Whitelist never consulted.

Scenario: A user whose email has been removed from `OAUTH_ALLOWED_USERS` can still authenticate via any previously registered Passkey. The whitelist is bypassed entirely.

**Root cause WHY:** The WebAuthn login was implemented as a separate code path from OAuth login. The developer duplicated the token-issuing logic but forgot to include the whitelist check.

**Meta-pattern:** "Parallel auth paths with incomplete security replication" -- when multiple authentication methods exist, each must apply all access control checks.

---

### S18: WebAuthn Credential Operations Not User-Scoped

**Interrogate:**
- *Why?* The credential ID is used as the sole identifier.
- *What does it do?* `deleteCredential` and `updateCredentialName` take a `credentialId` parameter from the URL and operate on it without checking if it belongs to the authenticated user.
- *What does it promise?* Users can only manage their own credentials.

**FALSIFY:**
At `webAuthnService.ts:216-230`:
```typescript
async updateCredentialName(credentialId: string, name: string) {
    await this.env.DB.update(schema.authPasskeys)
        .set({ name })
        .where(eq(schema.authPasskeys.credentialId, credentialId));
    // No check: credential.userId === currentUser
}

async deleteCredential(credentialId: string) {
    await this.env.DB.delete(schema.authPasskeys)
        .where(eq(schema.authPasskeys.credentialId, credentialId));
    // No check: credential.userId === currentUser
}
```

The route handlers at `authRoutes.ts:246-264` pass the credential ID from the URL directly to the service. The `authMiddleware` ensures a valid JWT, but the user from the JWT is not compared to the credential's owner.

However, in this application's design (S24), there appears to be a single-user or shared-user vault model. Let me check: the vault has no `userId` column, and all authenticated users share the same vault. If this is truly a single-user system, then the "other user" scenario doesn't apply.

But `authPasskeys.userId` stores email, and multiple users CAN register passkeys (via different OAuth providers). User A could delete User B's passkey.

Also, `listCredentials()` at line 201-210 lists ALL credentials across ALL users (no userId filter).

**Verdict: PROVEN**

**Proof:** `webAuthnService.ts:200-230` -- `listCredentials`, `updateCredentialName`, and `deleteCredential` operate on ALL passkeys in the database without filtering by the authenticated user's identity. Any authenticated user can:
1. List all registered passkeys (including other users').
2. Delete any passkey by ID.
3. Rename any passkey.

Input: `DELETE /api/oauth/webauthn/credentials/VICTIM_CREDENTIAL_ID` with a valid auth token for a different user.

**Root cause WHY:** Single-user assumption. The developer assumed only one user would exist, so credential scoping was unnecessary.

**Meta-pattern:** "Missing authorization check on object-level operations" -- IDOR (Insecure Direct Object Reference).

---

### S19: Arbitrary Filename/Content in Manual Backup

**Interrogate:**
- *Why?* The manual backup endpoint supports both frontend-generated and server-generated backup content.
- *What does it do?* When `filename` and `content` are both provided, they are passed directly to the backup provider.
- *What does it promise?* Backup operations are safe and validated.

**FALSIFY:**
At `backupService.ts:305-307`:
```typescript
if (filename && content) {
    finalFilename = filename;
    finalContent = content;
}
```

No `validateSafeFilename` is called on this path (unlike `downloadFile` and `deleteFile`). The `filename` is sent directly to the backup provider.

For WebDAV, the filename becomes part of a file path. If filename contains `../../../etc/passwd`, the WebDAV provider's `uploadBackup` constructs:
```
fullPath = saveDir + '/' + filename
```

However, the `uploadBackup` uses the webdav client's `putFileContents` which encodes the path, and the `getRequestUrl` function encodes each path segment. Path traversal through the filename would be server-dependent.

For S3, the filename becomes part of the object key. S3 allows `/` in keys but doesn't do directory traversal. Impact is limited to writing backup files with unusual names.

**Verdict: SUSPECTED**

The missing filename validation on the upload path is a real gap (the download/delete paths DO validate), but concrete exploitation depends on the backup provider's handling of special characters. WebDAV's path encoding likely prevents traversal.

---

### S20: Telegram Webhook Without Secret Verification

**Interrogate:**
- *Why?* The verification code is commented out with a "TODO" tone.
- *What does it do?* Any request to `/api/telegram/webhook` is processed as a Telegram update.
- *What does it promise?* Only Telegram's servers can trigger bot actions.

**FALSIFY:**
At `telegramRoutes.ts:11-13`:
```typescript
// 1. 安全校验 (可选，建议在 setWebhook 时设置 secret_token)
// const secretToken = c.req.header('X-Telegram-Bot-Api-Secret-Token');
// if (secretToken !== c.env.TELEGRAM_WEBHOOK_SECRET) return c.text('Unauthorized', 403);
```

An attacker can send a crafted POST to `/api/telegram/webhook` with a fake update. The endpoint only handles `/start` commands, which trigger a bot message to a chat ID. The attacker can:
1. Trigger the bot to send messages to arbitrary chat IDs (phishing).
2. The bot message includes a login URL with an attacker-controlled state parameter.

However, the login_url in the bot message uses the Telegram Login Widget protocol, which is cryptographically verified by Telegram. The attacker can't bypass the Telegram-side verification.

Impact: Minor -- attacker can make the bot send unsolicited messages, but can't compromise authentication.

**Verdict: SUSPECTED**

The missing webhook verification allows forged Telegram updates, but the security impact is limited to bot message spam. No authentication bypass is possible.

---

### S21: Health Check Exposes Security Config to Unauthenticated Users

**Interrogate:**
- *Why?* Health check is needed before login is possible.
- *What does it do?* Returns which security variables are misconfigured (names + levels).
- *What does it promise?* Useful diagnostics without information leakage.

**FALSIFY:**
The health check at `/api/health/health-check` has no auth. It returns:
```json
{
    "issues": [
        {"field": "OAUTH_GITHUB", "level": "error", "message": "github_config_incomplete", "missingFields": ["OAUTH_GITHUB_CLIENT_SECRET"]}
    ],
    "passedChecks": ["encryption_key_passed", "jwt_secret_passed", "oauth_allow_all_passed"]
}
```

An attacker learns:
- Which OAuth providers are configured.
- Whether encryption key and JWT secret meet length requirements.
- Whether OAUTH_ALLOW_ALL is enabled.
- Which specific env vars are missing.

This is information disclosure but not directly exploitable.

**Verdict: SUSPECTED**

The health endpoint leaks configuration status to unauthenticated users. While not directly exploitable, it helps attackers map the target's security posture.

---

### S22: Backup Config Encryption Fallback to JWT_SECRET

**Interrogate:**
- *Why?* Resilience -- if ENCRYPTION_KEY isn't set, fall back to JWT_SECRET.
- *What does it do?* Uses `this.env.ENCRYPTION_KEY || this.env.JWT_SECRET` as the encryption key for backup provider configs.
- *What does it promise?* Backup configs are always encrypted.

**FALSIFY:**
If `ENCRYPTION_KEY` is undefined/empty, `JWT_SECRET` is used instead. But the health check at `health.ts:27-41` blocks ALL API requests if `ENCRYPTION_KEY` is too short (< 32 chars). So in practice, the fallback to JWT_SECRET should never be reached in production.

However, the VaultService constructor at `vaultService.ts:17-19` throws if `ENCRYPTION_KEY` is missing, so vault operations fail. But backup service construction doesn't check this.

If somehow `ENCRYPTION_KEY` is set but empty string `''`, then `'' || JWT_SECRET` falls back to JWT_SECRET. But `''.length < 32` triggers the health check block.

**Verdict: HOLDS -> HARDENED**

The health check middleware at `index.ts:65-86` blocks all non-health API requests when `ENCRYPTION_KEY` is missing/short. The fallback to JWT_SECRET in backup service is dead code in practice.

---

### S23: PKCE codeVerifier Returned to Frontend

**Interrogate:**
- *Why?* PKCE requires the verifier to be sent during the token exchange.
- *What does it do?* The authorize endpoint returns `codeVerifier` in the JSON response. The frontend stores it and sends it back during callback.
- *What does it promise?* PKCE provides protection against authorization code interception.

**FALSIFY:**
In standard PKCE, the code verifier should be stored on the client that initiated the flow and sent during token exchange. Here, the backend generates the verifier, sends it to the frontend, and the frontend sends it back during callback. The backend doesn't store it (no server-side session).

This means the PKCE verifier travels: backend -> frontend (JSON) -> frontend -> backend (callback POST). The verifier is exposed in both the authorize response AND the callback request. If an attacker intercepts the authorize response, they have the verifier AND can intercept the callback code.

However, PKCE's primary threat model is authorization code interception by a different app on the same device (mobile). In a web context over HTTPS, intercepting the authorize response means the attacker already has a MITM position, making PKCE moot.

For Cloudflare Access specifically (the only provider using PKCE here), the verifier provides marginal security since the code exchange happens server-side.

**Verdict: SUSPECTED**

The PKCE verifier's round-trip through the frontend diminishes its security value, but the practical impact is minimal in a web HTTPS context.

---

### S24: No Per-User Vault Scoping

**Interrogate:**
- *Why?* This appears to be a single-user 2FA management system.
- *What does it do?* All vault operations read/write the same table without user ID filtering. Any authenticated user sees all secrets.
- *What does it promise?* Only authorized users access the vault.

**FALSIFY:**
The vault table has `created_by` column but NO queries filter by it. `findAll()`, `findPaginated()`, `create()`, `delete()` -- none use a user identifier in their WHERE clause.

If `OAUTH_ALLOW_ALL` is enabled, or if multiple users are whitelisted, ALL users see ALL secrets. This is by design for a personal 2FA manager, but the whitelist supports multiple entries.

Combined with S06 (OAUTH_ALLOW_ALL=2 bypass), S17 (WebAuthn skips whitelist), and S10 (Telegram always fails), the shared vault model means any access control bypass grants full secret access.

**Verdict: SUSPECTED**

This is a design decision (single-user vault) rather than a bug. However, the implication is that any auth bypass (S06, S17) immediately yields all 2FA secrets, not just the attacker's own data.

---

### S25: Relaxed Validation in add-from-uri

**Interrogate:**
- *Why?* QR code scanning may produce non-standard URIs.
- *What does it do?* Accepts any secret as long as it's non-empty (no `validateBase32Secret` check).
- *What does it promise?* Valid 2FA entries are stored.

**FALSIFY:**
At `vaultRoutes.ts:137-148`, the `add-from-uri` endpoint only checks `if (secret)`, not `validateBase32Secret(secret)`. But the `createAccount` service method at `vaultService.ts:86` DOES call `validateBase32Secret(secret)`.

So the relaxed parsing at the route level is followed by strict validation in the service layer. The route-level parsing allows extraction of data from non-standard URIs, but the service rejects invalid secrets.

**Verdict: HOLDS -> HARDENED**

The service layer's `validateBase32Secret` check catches invalid secrets regardless of route-level parsing leniency.

---

## Chained Findings

From S04 PROVEN (CF-Connecting-IP spoofing in Docker), grepping for other uses of `CF-Connecting-IP`:

The header is used in rate limit key construction (S04 covers all paths). No other uses found.

From S05 PROVEN (path mismatch), grepping for other hardcoded path strings:
- `authRoutes.ts:225`: WebAuthn reset also uses wrong prefix `/api/auth/` -- same bug, same pattern.

From S17 PROVEN (WebAuthn skips whitelist), examining what happens with credential registration:
- `authRoutes.ts:170`: Registration requires `authMiddleware` but doesn't verify the authenticated user matches the `userEmail` passed to registration. However, the `user.email` is extracted from the JWT which is server-issued, so this is not directly exploitable.

---

## 1. Proof Catalog

| # | Finding | Verdict | Root Cause | Meta-Pattern |
|---|---------|---------|------------|--------------|
| S04 | CF-Connecting-IP spoofable in Docker mode bypasses rate limiting | PROVEN | Assumed deployment always behind Cloudflare edge | Deployment environment assumptions |
| S05 | Rate limit reset key uses `/api/auth/` but actual path is `/api/oauth/` -- counters never cleared | PROVEN | Path prefix renamed without updating hardcoded strings | Hardcoded strings diverge from routing |
| S06 | OAUTH_ALLOW_ALL=2 bypasses whitelist but passes health check | PROVEN | `'2'` added to auth check but not health check | Security checks with divergent logic |
| S07/S08 | ENCRYPTION_KEY (master key) sent in API response during first-time setup | PROVEN | Needed for UI display, embedded in standard API responses | Secrets in regular API responses |
| S10 | Telegram login always rejected (whitelistFields=['id'] but checker only handles 'email'/'username') | PROVEN | Whitelist system not extended for Telegram's identity model | Extension point not wired to all dimensions |
| S14 | Export password minimum length reduced from 12 to 5 by local variable shadowing | PROVEN | Dev shortcut/testing code left in production | Local variable shadows security config |
| S17 | WebAuthn Passkey login completely skips whitelist verification | PROVEN | Separate code path didn't replicate all security checks | Parallel auth paths with incomplete replication |
| S18 | WebAuthn credential CRUD not scoped to authenticated user (IDOR) | PROVEN | Single-user assumption; no userId filter | Missing object-level authorization |

## 2. Suspicion List

| # | Finding | What Was Tried |
|---|---------|----------------|
| S03 | Rate limiter silent skip on DB error | Traced all catch paths; confirmed silent proceed on DB errors. Impact limited due to OAuth-only auth model. |
| S09 | Telegram 24-hour auth_date window | Verified window calculation. Replay requires hash interception in specific scenario. |
| S11 | Logout without CSRF protection | Confirmed no authMiddleware on logout route. Impact is forced logout only. |
| S12 | postMessage with wildcard origin in backup OAuth callbacks | State cookie prevents direct exploitation. Requires XSS for concrete impact. |
| S13 | Template literal XSS in backup OAuth error messages | Confirmed unsanitized interpolation in inline script. Data source (OAuth provider responses) not attacker-controlled. |
| S16 | `raw` import type outside try-catch | Confirmed code path. Validation still runs post-parse. Auth required. |
| S19 | Arbitrary filename in manual backup upload | Confirmed missing validateSafeFilename. Provider-level encoding likely prevents traversal. |
| S20 | Telegram webhook without secret_token | Confirmed commented-out verification. Impact limited to bot message spam. |
| S21 | Health check exposes config status to unauthenticated users | Confirmed information disclosure. No direct exploitation path. |
| S23 | PKCE verifier round-trips through frontend | Confirmed verifier exposure. Marginal impact in HTTPS web context. |
| S24 | Shared vault with no per-user scoping | Confirmed design choice. Amplifies impact of any auth bypass. |

## 3. Defense Map

| # | Defense | Attacks Survived |
|---|---------|-----------------|
| S01 | CORS origin reflection + CSRF Double Submit Cookie | Cross-origin credentialed requests blocked because attacker cannot read csrf_token cookie cross-origin. Set-Cookie header not accessible via fetch. document.cookie is same-origin only. |
| S02 | CSRF Double Submit Cookie pattern | All mutating endpoints except logout pass through authMiddleware which verifies cookie+header match. |
| S15 | Drizzle ORM parameterized LIKE queries | SQL injection attempted via search parameter. ORM parameterizes values. LIKE wildcard injection has no security impact since users already have full access. |
| S22 | Backup config encryption fallback guarded by health check | Health check blocks requests when ENCRYPTION_KEY is missing, making the JWT_SECRET fallback unreachable. |
| S25 | Service-layer validateBase32Secret after relaxed route parsing | Invalid secrets caught by service layer regardless of route-level leniency. |

## 4. Root Causes

### RC1: Parallel Code Paths Without Security Replication
**Shared by:** S10, S17, S18

Multiple authentication and authorization paths were implemented independently. The OAuth flow has whitelist checks and user-scoped operations, but the WebAuthn flow and Telegram integration were added without replicating all security controls. This is the highest-impact root cause, as it allows complete access control bypass.

### RC2: Hardcoded Strings Diverging from Actual Configuration
**Shared by:** S05, S06

When paths or configuration values are referenced by hardcoded strings in multiple locations, renaming or extending values in one location without updating all references creates security gaps. The rate limit reset path mismatch and the OAUTH_ALLOW_ALL check divergence share this root cause.

### RC3: Deployment Environment Assumptions
**Shared by:** S04, S03

Security mechanisms that depend on infrastructure guarantees (Cloudflare edge stripping headers, D1 database availability) silently fail when the application runs in alternative deployment modes (Docker). The code has no runtime detection or fallback for non-Cloudflare environments.

### RC4: Secrets in API Responses
**Shared by:** S07, S08

The master encryption key is transmitted in standard API response bodies, creating multiple exposure vectors. Secrets should use dedicated, auditable, one-time display channels rather than regular API patterns.

### RC5: Development Artifacts in Production
**Shared by:** S14, S16

Local testing shortcuts (reduced password length, undocumented import types) were left in production code. These directly weaken security guarantees that the global configuration promises.

## 5. Seed Map

| Metric | Count |
|--------|-------|
| Total Seeds | 25 |
| PROVEN | 8 |
| SUSPECTED | 11 |
| HARDENED | 5 |
| UNCLEAR | 1 (S24 - design decision) |
| **Falsification success rate** | **32%** (8/25) |

## 6. Unexplored

| # | Item | Reason |
|---|------|--------|
| S24 | Shared vault model as design decision | Cannot falsify a design choice; documented as amplifier for other findings |
| Frontend serialization strategies | aegisStrategy.js, protonPassStrategy.js, etc. not deep-audited | Lower priority; behind auth wall, data flows from trusted local state |
| Backup providers (Telegram, Google Drive, OneDrive, Baidu, Dropbox) | Full provider interaction flows not audited for SSRF, token handling | Each provider warrants its own deep audit |
| Frontend IndexedDB caching | device_salt, vault data cached locally | Client-side persistence security not in scope of backend-focused audit |
