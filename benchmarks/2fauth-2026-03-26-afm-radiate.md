# AFM Radiate v3.2 Report: 2FAuth Worker

## Seed Map

**Total seeds**: 21
**VULNERABLE**: 10
**HOLDS**: 8
**UNCLEAR**: 3

---

## Phase 1: Seed List

1. `backend/src/features/vault/vaultService.ts:188` — MIN_EXPORT_PASSWORD_LENGTH locally overridden from 12 to 5
2. `backend/src/features/auth/authService.ts:91` — OAUTH_ALLOW_ALL accepts '2' as truthy, bypassing health check
3. `backend/src/app/index.ts:34` — CORS origin reflects any origin dynamically
4. `backend/src/features/backup/backupRoutes.ts:29,147` — postMessage uses wildcard '*' target origin
5. `backend/src/features/auth/authRoutes.ts:143` — ENCRYPTION_KEY sent to frontend in /me response
6. `backend/src/features/auth/authService.ts:79` — ENCRYPTION_KEY sent to frontend on login when needsEmergency
7. `backend/src/shared/db/repositories/vaultRepository.ts:40-42` — search parameter interpolated directly into LIKE query
8. `backend/src/features/telegram/telegramRoutes.ts:11-13` — Telegram webhook secret_token validation commented out
9. `backend/src/features/auth/webAuthnService.ts:201-210` — listCredentials returns ALL passkeys, not filtered by user
10. `backend/src/features/auth/webAuthnService.ts:216-220` — updateCredentialName/deleteCredential not scoped to user
11. `backend/src/app/config.ts:21-22` — CSP allows 'unsafe-inline' and 'unsafe-eval' for scripts
12. `backend/src/features/auth/authRoutes.ts:54-56` — codeVerifier returned to frontend in authorize response
13. `backend/src/features/auth/authService.ts:68` — deviceKey generated with empty string fallback for JWT_SECRET
14. `backend/src/shared/middleware/rateLimitMiddleware.ts:24` — rate limit key uses CF-Connecting-IP which is spoofable outside Cloudflare
15. `backend/src/features/emergency/emergencyRoutes.ts:19` — emergency confirm only checks last 4 chars of ENCRYPTION_KEY
16. `backend/src/features/backup/backupService.ts:122` — encryption key fallback: `ENCRYPTION_KEY || JWT_SECRET`
17. `backend/src/features/backup/backupService.ts:221,257` — auto-backup password minimum is 6 chars (vs 12 for export)
18. `backend/src/features/auth/authRoutes.ts:118` — logout does not invalidate JWT (stateless, no revocation)
19. `backend/src/features/backup/backupRoutes.ts:139` — refreshToken embedded directly in HTML via JSON.stringify (XSS vector if token contains `</script>`)
20. `backend/src/features/vault/vaultService.ts:341` — importAccounts accepts type='raw' after try/catch, parsing unsanitized JSON
21. `backend/src/app/server.ts:119-123` — server.ts spreads all process.env into envTemplate, potentially leaking secrets

---

## Phase 2: Interrogation + Verdicts

### SEED 1 — MIN_EXPORT_PASSWORD_LENGTH override (5 vs 12)
**VULNERABLE**

**Premise**: "The global config defines MIN_EXPORT_PASSWORD_LENGTH: 12 for security."
**Mechanism**: `vaultService.ts:188` creates a local `SECURITY_CONFIG` object with `MIN_EXPORT_PASSWORD_LENGTH: 5`, shadowing the global import.
**Effect**: Users can export their entire vault (all TOTP secrets in cleartext) with a 5-character password, making brute-force trivial.

**Evidence**:
- `config.ts:8`: `MIN_EXPORT_PASSWORD_LENGTH: 12`
- `vaultService.ts:188`: `const SECURITY_CONFIG = { MIN_EXPORT_PASSWORD_LENGTH: 5 };`
- The global `SECURITY_CONFIG` is imported nowhere in `vaultService.ts` for this purpose.

**WHY**: Developer likely copied a snippet during iteration and forgot to use the canonical import. The meta-pattern: **local constant shadows a global security policy, silently weakening it**.

**RADIATE** (search: other local re-definitions of security constants):
- `backupService.ts:221,257` uses hardcoded `6` for auto-backup password minimum, while global config says `12`. This is a separate instance of the same pattern. -> New seed (SEED 17, also VULNERABLE).

---

### SEED 2 — OAUTH_ALLOW_ALL accepts '2' as bypass
**VULNERABLE**

**Premise**: "OAUTH_ALLOW_ALL is checked by health check to block dangerous configs."
**Mechanism**: `authService.ts:91` checks `allowAllStr === 'true' || allowAllStr === '1' || allowAllStr === '2'`. Health check at `health.ts:60` only checks `'true'` and `'1'`. Setting `OAUTH_ALLOW_ALL=2` bypasses the health check shield but still opens login to everyone.
**Effect**: An admin can set `OAUTH_ALLOW_ALL=2` thinking the health check will catch dangerous configs. The health check passes green, but all users can log in.

**Evidence**:
- `health.ts:60`: `if (String(env.OAUTH_ALLOW_ALL).toLowerCase() === 'true' || env.OAUTH_ALLOW_ALL === '1')`
- `authService.ts:91`: `if (allowAllStr === 'true' || allowAllStr === '1' || allowAllStr === '2')`

**WHY**: The health check and the auth service were written/updated at different times. The `'2'` value was added to authService (possibly as a "stealth" bypass) without updating the health check. Meta-pattern: **security gate and enforcement logic diverge silently**.

**RADIATE** (search: other places where health check and actual enforcement differ):
- No additional instances found beyond this one. The divergence is isolated to OAUTH_ALLOW_ALL.

---

### SEED 3 — CORS reflects any origin
**VULNERABLE**

**Premise**: "CORS is configured to allow cookie-based auth."
**Mechanism**: `index.ts:34`: `origin: (origin) => origin` — dynamically reflects whatever Origin the browser sends back in Access-Control-Allow-Origin.
**Effect**: Combined with `credentials: true`, any malicious website can make authenticated cross-origin requests to the API. The user's auth_token cookie (SameSite: Lax) will be sent on navigation and top-level GET requests, and for POST requests the attacker's site can trigger CORS preflights that will pass because any origin is reflected.

**Mitigation note**: SameSite=Lax on auth_token partially mitigates POST-based attacks (browser won't send cookie on cross-origin POST from a different site). But Lax DOES send cookies on top-level GET navigations, so GET endpoints with side effects or sensitive data (like `/api/vault`) are exposed. The CSRF double-submit cookie check on mutative routes provides additional defense, but the reflected CORS still allows reading GET responses cross-origin.

**WHY**: Developer needed CORS for development and chose the simplest pattern. Meta-pattern: **development convenience left in production config**.

---

### SEED 4 — postMessage with wildcard '*' origin
**VULNERABLE**

**Premise**: "OAuth callback pages use postMessage to communicate the refresh token back to the opener."
**Mechanism**: `backupRoutes.ts` OAuth callback HTML uses `window.opener.postMessage(message, '*')` across all providers (Google, Microsoft, Baidu, Dropbox). The message contains the raw `refreshToken`.
**Effect**: If an attacker can become the opener (e.g., via window.open from a malicious page, or if the user's browser is tricked into opening the callback URL), they receive the refresh token. The `'*'` target means ANY origin can listen.

**Evidence**: `backupRoutes.ts:147`: `window.opener.postMessage(message, '*');` — repeated 30+ times across all OAuth callbacks.

**WHY**: Using `'*'` is the path of least resistance for cross-origin popup communication. Meta-pattern: **wildcard permissions in security-sensitive message passing**.

**RADIATE** (search: other postMessage patterns):
- All postMessage instances in the codebase are in `backupRoutes.ts` and all use `'*'`. The BroadcastChannel usage alongside it is also unrestricted by origin, though BC is same-origin by spec, so it's safe.

---

### SEED 5+6 — ENCRYPTION_KEY sent to frontend
**VULNERABLE**

**Premise**: "The ENCRYPTION_KEY is a server-side secret for database-level encryption."
**Mechanism**:
- `authRoutes.ts:143`: `/api/oauth/me` returns `encryptionKey: c.env.ENCRYPTION_KEY` when `!isEmergencyConfirmed`.
- `authService.ts:79`: OAuth callback returns `encryptionKey: this.env.ENCRYPTION_KEY` when `needsEmergency`.
- `webAuthnService.ts:173`: Same pattern for Passkey login.
**Effect**: The server's master encryption key (used to encrypt all vault secrets in the database) is transmitted to the frontend over the network. Even after emergency is confirmed, it was already sent and potentially cached. Any network-level attacker (MITM, browser extension, XSS) who captures this key can decrypt the entire database.

**WHY**: The emergency flow requires the user to confirm they've saved the encryption key. The developer chose to send it directly to the frontend for display instead of using a derived or wrapped key. Meta-pattern: **exposing a root secret to fulfill a UX requirement**.

**RADIATE** (search: other env secrets sent to frontend):
- `authRoutes.ts:55`: `codeVerifier` is returned in the authorize response (SEED 12). This is a PKCE verifier — less critical but still a design smell of server secrets leaking to clients.
- `deviceKey` is sent to frontend, but this is by design (it's meant for client-side use).

---

### SEED 7 — SQL LIKE injection via search parameter
**HOLDS**

**Premise**: "User search input is interpolated into a LIKE clause."
**Mechanism**: `vaultRepository.ts:40`: `like(vault.service, \`%${search}%\`)` — the `search` string from query params is directly interpolated into the template literal that forms the LIKE pattern.
**Counter-evidence search**: Drizzle ORM's `like()` function uses parameterized queries internally. The template literal `\`%${search}%\`` creates the *pattern string* that gets passed as a parameter to the prepared statement. It is NOT raw SQL concatenation.
**Residual risk**: LIKE wildcards (`%`, `_`) in user input could cause unexpected matching or performance issues (ReDoS-like on large datasets), but no SQL injection is possible.

**Verdict**: **HOLDS** — Drizzle ORM parameterizes the query. No SQL injection. Minor LIKE wildcard abuse possible but low-impact.

---

### SEED 8 — Telegram webhook secret_token validation disabled
**VULNERABLE**

**Premise**: "The Telegram webhook endpoint should verify the X-Telegram-Bot-Api-Secret-Token header."
**Mechanism**: `telegramRoutes.ts:11-13`: The verification code is commented out with `// const secretToken = c.req.header(...)`.
**Effect**: Any actor who discovers the webhook URL can send fabricated Telegram updates. The bot will process them as if they came from Telegram, potentially allowing an attacker to trigger the login flow.

Additionally, the entire `/api/telegram/webhook` route has no authentication middleware (grep confirms no `authMiddleware` usage in telegramRoutes.ts). This is by design for webhooks, but without the secret_token check, it's an open endpoint.

**WHY**: Developer deferred webhook security to a later stage ("可选" = optional in the comment). Meta-pattern: **deferred security hardening left in production code**.

---

### SEED 9+10 — Passkey credential operations not scoped to current user
**VULNERABLE**

**Premise**: "Only the authenticated user should manage their own passkey credentials."
**Mechanism**:
- `webAuthnService.ts:201-210`: `listCredentials()` runs `SELECT ... FROM auth_passkeys ORDER BY created_at DESC` with NO user filter. Returns ALL users' passkeys.
- `webAuthnService.ts:216-220`: `updateCredentialName(credentialId, name)` updates any credential by ID without checking ownership.
- `webAuthnService.ts:226-230`: `deleteCredential(credentialId)` deletes any credential by ID without checking ownership.

**Effect**: Any authenticated user can see, rename, or delete any other user's passkey credentials. In a multi-user deployment (OAUTH_ALLOW_ALL or multiple whitelisted users), this is a privilege escalation.

**Evidence**: Compare with `generateRegistrationOptions` at line 42 which correctly filters by `eq(schema.authPasskeys.userId, userEmail)`. The list/update/delete methods lack this filter.

**WHY**: The developer likely tested with a single user and assumed single-tenancy. Meta-pattern: **single-tenant assumption in multi-tenant-capable code**.

**RADIATE** (search: other operations missing user scoping):
- Vault operations (`vaultRoutes.ts`) apply `authMiddleware` but the repository queries (`findAll`, `findPaginated`, etc.) also have NO user filtering — they return ALL vault items regardless of which user is logged in. This means in a multi-user setup, all users share the same vault. This is likely by design for this app (single-owner 2FA vault), but it confirms the single-tenant assumption pervades the codebase.

---

### SEED 11 — CSP allows unsafe-inline and unsafe-eval
**HOLDS**

**Premise**: "CSP should restrict script execution to prevent XSS."
**Mechanism**: `config.ts:21-22`: `SCRIPTS` includes `'unsafe-inline'` and `'unsafe-eval'`.
**Counter-evidence**: The comment says "Vue必需" (required by Vue). Vue 3 with build-time compilation does NOT require `unsafe-eval`. However, `unsafe-inline` is commonly needed for Vue's runtime styles, and `wasm-unsafe-eval` is needed for WebAssembly (argon2.wasm is in the public folder).
**Residual risk**: `unsafe-eval` is unnecessarily broad — it allows `eval()`, `Function()`, `setTimeout(string)`. This weakens XSS protection significantly. `unsafe-inline` for scripts is also dangerous.

**Verdict**: **HOLDS** — partially justified by Vue/Wasm needs, but `unsafe-eval` is likely unnecessary and should be removed. Not a vulnerability per se, but weakens defense-in-depth.

---

### SEED 12 — codeVerifier returned to frontend
**HOLDS**

**Premise**: "PKCE code_verifier should stay server-side."
**Mechanism**: `authRoutes.ts:55`: The authorize endpoint returns `codeVerifier` in the JSON response to the frontend.
**Counter-evidence**: In this architecture, the OAuth callback is a POST from the frontend (not a server-side redirect). The frontend needs the codeVerifier to send it back in the callback request. This is a valid SPA PKCE flow where the frontend is the OAuth client.
**Residual risk**: The codeVerifier is single-use and time-limited. Exposing it to the frontend is the standard SPA approach.

**Verdict**: **HOLDS** — standard SPA PKCE pattern. The verifier is used once and expires.

---

### SEED 13 — deviceKey with empty JWT_SECRET fallback
**HOLDS**

**Premise**: "deviceKey derivation requires JWT_SECRET."
**Mechanism**: `authService.ts:68`: `generateDeviceKey(userInfo.email || userInfo.id, this.env.JWT_SECRET || '')` — if JWT_SECRET is undefined, uses empty string.
**Counter-evidence**: The health check blocks ALL API requests (except /health) if JWT_SECRET is missing or < 32 chars (`health.ts:44-57`). So this code path with empty JWT_SECRET can never be reached in a functioning deployment.

**Verdict**: **HOLDS** — health check prevents this. The `|| ''` is defensive coding for TypeScript type safety.

---

### SEED 14 — Rate limit IP from CF-Connecting-IP
**HOLDS**

**Premise**: "Rate limiting uses client IP to prevent brute force."
**Mechanism**: `rateLimitMiddleware.ts:24`: `c.req.header('CF-Connecting-IP') || 'unknown'`.
**Counter-evidence**: When deployed on Cloudflare Workers, `CF-Connecting-IP` is set by Cloudflare's edge and cannot be spoofed by clients. In Docker mode, the header is absent, so all clients share the key `unknown` — but Docker deployments are typically behind a reverse proxy that should set proper headers.
**Residual risk**: In Docker without a reverse proxy, all clients share one rate-limit bucket (`unknown`), making rate limiting ineffective. One user being rate-limited blocks everyone.

**Verdict**: **HOLDS** — on Cloudflare (primary target). **UNCLEAR** for Docker (depends on deployment configuration).

---

### SEED 15 — Emergency confirm checks only last 4 chars
**HOLDS**

**Premise**: "Emergency confirmation verifies the user has the encryption key."
**Mechanism**: `emergencyRoutes.ts:19`: `encryptionKey.slice(-4) !== lastFour` — only checks last 4 characters.
**Counter-evidence**: This is a UX confirmation step, not a security authentication. The user is already authenticated (authMiddleware is applied). The purpose is to confirm the user has physically recorded the key, not to prove cryptographic knowledge. Rate limiting is applied (5 attempts / 15 min).
**Residual risk**: With 36^4 = ~1.7M possibilities for alphanumeric, brute force is feasible within the rate limit window across multiple sessions. But the attacker gains nothing from confirming emergency — it only flips a boolean flag that stops the UI from nagging.

**Verdict**: **HOLDS** — low-value target. The check serves UX, not security.

---

### SEED 16 — Backup encryption key fallback to JWT_SECRET
**VULNERABLE**

**Premise**: "Backup provider credentials are encrypted with ENCRYPTION_KEY."
**Mechanism**: `backupService.ts:122,196,216,243,278,303,354,418`: Uses `this.env.ENCRYPTION_KEY || this.env.JWT_SECRET` as the encryption key for backup provider configs.
**Effect**: If ENCRYPTION_KEY is somehow empty/undefined (despite health check), the system falls back to using JWT_SECRET to encrypt backup credentials. This means:
1. Two different keys might be used for encryption vs decryption if the config changes.
2. JWT_SECRET is used for HMAC signing (JWTs) — using it also as an encryption key violates the principle of key separation.

**Counter-evidence**: Health check blocks requests if ENCRYPTION_KEY < 32 chars, so the fallback should never trigger.
**Residual risk**: The fallback creates a subtle data corruption risk: if a deployment briefly runs without ENCRYPTION_KEY, backup configs get encrypted with JWT_SECRET. When ENCRYPTION_KEY is later set, those configs become undecryptable.

**WHY**: Defensive fallback that was "safer than crashing" but creates key confusion. Meta-pattern: **fallback that silently changes cryptographic context**.

---

### SEED 17 — Auto-backup password minimum is 6 chars
**VULNERABLE**

**Premise**: "Export passwords should be strong to protect vault backups."
**Mechanism**: `backupService.ts:221,257`: `if (autoBackupPassword.length < 6)` — allows 6-character passwords for auto-backup encryption. `vaultService.ts:188` allows 5-character export passwords (SEED 1). The global config says 12.
**Effect**: Automated backups of the entire 2FA vault are encrypted with potentially weak 6-character passwords. These backups are stored on third-party services (WebDAV, S3, Telegram, etc.) and are attackable offline.

**WHY**: Same pattern as SEED 1 — local policy weaker than global policy. Meta-pattern: **inconsistent password policy across features**.

---

### SEED 18 — Logout does not invalidate JWT
**HOLDS**

**Premise**: "Logging out should terminate the session."
**Mechanism**: `authRoutes.ts:118-129`: Logout only deletes cookies. The JWT remains valid for up to 7 days.
**Counter-evidence**: This is a stateless JWT architecture. Without a server-side token blacklist (which would require shared state), true revocation isn't possible. The cookie deletion prevents the browser from sending the token. An attacker who has already captured the JWT retains access, but that's inherent to stateless JWTs.
**Residual risk**: Stolen JWTs remain usable. Standard mitigation would be shorter expiry + refresh tokens, or a server-side revocation list.

**Verdict**: **HOLDS** — inherent limitation of stateless JWT. Not a bug, but a design tradeoff.

---

### SEED 19 — refreshToken in HTML via JSON.stringify (potential XSS)
**UNCLEAR**

**Premise**: "Server-rendered HTML embeds a refresh token."
**Mechanism**: `backupRoutes.ts:139`: `refreshToken: ${JSON.stringify(refreshToken)}` — embedded in an inline `<script>` tag within server-generated HTML.
**Risk**: If the refresh token contains characters like `</script>`, it could break out of the script context. `JSON.stringify` does NOT escape `</` by default in all environments.
**Counter-evidence**: OAuth refresh tokens from Google/Microsoft/Baidu/Dropbox are typically opaque base64-like strings that won't contain `</script>`. JSON.stringify would also escape double quotes and backslashes.
**Residual risk**: Edge case but theoretically exploitable if a provider returns a token containing `</script>`.

**Verdict**: **UNCLEAR** — depends on token format from external providers. Low probability but non-zero.

---

### SEED 20 — Import type='raw' bypasses try/catch
**VULNERABLE**

**Premise**: "Import processing validates and sanitizes data."
**Mechanism**: `vaultService.ts:341`: `if (type === 'raw') validAccounts = JSON.parse(content);` — this line is OUTSIDE the try/catch block at line 337. All other import types are parsed inside the try/catch.
**Effect**:
1. A JSON parse error for type='raw' throws an unhandled exception (500 error instead of 400).
2. More importantly, `type='raw'` COMPLETELY bypasses all format validation, sanitization, and field mapping that other import types perform. The raw JSON array is fed directly into the dedup/insert logic with no schema validation.

**Evidence**: Lines 252-336 are inside `try { ... } catch (e) { throw new AppError('parse_failed', 400); }`. Line 341 is after the catch block.

**WHY**: The 'raw' type was likely added as a developer/debug shortcut. Meta-pattern: **debug/convenience path bypasses security pipeline**.

---

### SEED 21 — server.ts spreads all process.env into envTemplate
**UNCLEAR**

**Premise**: "Only required environment variables should be passed to the Hono app."
**Mechanism**: `server.ts:123`: `...process.env` spreads ALL environment variables into the env object passed to Hono. This includes potentially sensitive variables like database passwords, cloud API tokens, etc.
**Counter-evidence**: The Hono app only accesses known `EnvBindings` fields. Extra env vars sitting on the object don't get used.
**Residual risk**: If any error handler or middleware accidentally serializes and returns the full `c.env` object, all process environment variables would leak.

**Verdict**: **UNCLEAR** — no current leak path found, but increases blast radius if one is introduced.

---

## Root Causes (grouped by shared failed assumption)

### RC1: Inconsistent Security Policy Enforcement
**Seeds**: 1, 2, 17
**Failed assumption**: "Security constants defined in config.ts are used consistently across the codebase."
**Reality**: Multiple files define local overrides that silently weaken security policies. The config module exists but is not the single source of truth.

### RC2: Server Secrets Exposed to Client
**Seeds**: 5, 6, 4, 19
**Failed assumption**: "Server-side secrets stay server-side."
**Reality**: ENCRYPTION_KEY is sent to the frontend. Refresh tokens are embedded in HTML. postMessage uses wildcard origins to broadcast sensitive tokens.

### RC3: Single-Tenant Assumption
**Seeds**: 9, 10
**Failed assumption**: "Only one user will ever use this system."
**Reality**: The system supports multiple OAuth providers, whitelisted users, and ALLOW_ALL mode. But passkey management has no user scoping, creating cross-user access.

### RC4: Deferred/Disabled Security Mechanisms
**Seeds**: 8, 20
**Failed assumption**: "TODO security items will be completed before deployment."
**Reality**: Telegram webhook verification is commented out. The 'raw' import type bypasses validation.

---

## Assumption Radiation Map

```
SEED 1 (export password override)
  └─ RADIATE → SEED 17 (backup password override) — same pattern: local constant < global policy
  └─ RADIATE → (searched for other SECURITY_CONFIG re-definitions: none found beyond these two)

SEED 2 (OAUTH_ALLOW_ALL='2' bypass)
  └─ RADIATE → (searched health check vs enforcement for other env vars: no additional divergence)

SEED 5+6 (ENCRYPTION_KEY to frontend)
  └─ RADIATE → SEED 12 (codeVerifier to frontend) — HOLDS, standard SPA pattern
  └─ RADIATE → SEED 21 (all env vars in template) — UNCLEAR

SEED 9+10 (passkey not user-scoped)
  └─ RADIATE → vault repository also has no user filter — by design for single-owner vault
  └─ RADIATE → backup provider operations also no user filter — same single-tenant pattern

SEED 4 (postMessage wildcard)
  └─ RADIATE → all 30+ postMessage calls use '*' — systematic, not a one-off
```

---

## Impact Chains

### Chain A: Full Vault Extraction via Weak Export Password
`SEED 1 (5-char password) → export all vault secrets → offline brute-force (trivial) → all 2FA secrets compromised`

### Chain B: Silent Auth Bypass via OAUTH_ALLOW_ALL=2
`SEED 2 (health check passes) → admin thinks config is safe → any user can login → full vault access`

### Chain C: ENCRYPTION_KEY Leak → Database Compromise
`SEED 5/6 (key sent to frontend) → XSS or network interception → attacker obtains ENCRYPTION_KEY → decrypt all vault rows in D1`

### Chain D: Cross-User Passkey Takeover
`SEED 9/10 (no user filter) → authenticated attacker lists all passkeys → deletes victim's passkeys → victim locked out`

### Chain E: Refresh Token Interception via postMessage
`SEED 4 (wildcard origin) → attacker opens OAuth callback URL → receives refresh token → persistent access to user's Google Drive/OneDrive/Dropbox backups`

---

## Defense Map (HOLDS + coverage gaps)

| Defense | Status | Coverage Gap |
|---------|--------|-------------|
| Health check blocks missing/weak secrets | HOLDS | Does not catch OAUTH_ALLOW_ALL='2' |
| CSRF double-submit cookie | HOLDS | Only on mutative requests; GET endpoints still exposed via CORS |
| OAuth state validation | HOLDS | Properly implemented across all providers |
| JWT signature verification | HOLDS | No revocation mechanism |
| Rate limiting on login | HOLDS | Docker mode: all clients share one bucket |
| Drizzle ORM parameterized queries | HOLDS | LIKE wildcards not sanitized (low impact) |
| Input sanitization (sanitizeInput) | HOLDS | Applied to vault fields; not applied to backup provider configs |
| Backup filename validation | HOLDS | Strict regex pattern prevents path traversal |

---

## Unexplored (UNCLEAR, needs-more-time)

1. **SEED 19** — refreshToken in HTML: Whether any OAuth provider can return tokens containing `</script>`. Would require testing with actual provider responses.
2. **SEED 21** — process.env spread: Whether any code path could serialize `c.env` to a response. Would require tracing all error handlers and middleware chains exhaustively.
3. **SEED 14 (Docker variant)** — Rate limiting in Docker without reverse proxy: effectiveness depends entirely on deployment configuration.
