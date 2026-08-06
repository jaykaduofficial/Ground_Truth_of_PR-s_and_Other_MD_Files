# PR Review: lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR11059__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR11059__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR11059__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR11059__20260430@pr:1`
- **Files changed:** 40
- **Route:** code_pr_ensemble
- **Reviewed:** 7/7/2026, 8:45:43 PM

## Metrics

- **Findings:** 4 unique (10 raw) · **Files flagged:** 4 · **Density:** 0.1 findings/file
- **Severity:** critical 0 · high 3 · medium 1
- **Files changed:** 40
- **Route:** code_pr_ensemble
- **By category:** security 2 · general 1 · authz 1
- **Top files:** app-credential.ts (1), parseRefreshTokenResponse.ts (1), refreshOAuthTokens.ts (1), CalendarService.ts (1)
- **Sources:** lens 0 · llm 10 · merged 4
- **Duplicates merged:** 6

## Summary

PR introduces syncing app credentials between Cal.com and self-hosted platforms, but there are several correctness and security issues to address. Webhook header validation is brittle due to header key lowercasing/configurability, and the refresh-token response parsing uses an invalid Zod schema that won’t properly validate/catch unknown fields. Token refresh handling appears inconsistent (Google credentials updated from `res.data` may be missing when using the sync endpoint), and the sync fetch needs hardening (set `Content-Type`, check `response.ok`, and add authentication).

## Findings

### HIGH · security

- **Location:** `apps/web/pages/api/webhook/app-credential.ts:20–40`
- **Lens:** llm
- **Rationale:** Header lookup is brittle: Node/Next lowercases header keys, and `process.env.CALCOM_WEBHOOK_HEADER_NAME` may be configured with uppercase/mixed case. Also, `req.headers[...]` can be `string | string[] | undefined`, so strict comparison to a string env var may fail unexpectedly.
- **Suggestion:** Normalize the header name to lowercase and coerce the header value safely: `const headerName=(process.env...||'calcom-webhook-secret').toLowerCase(); const provided = Array.isArray(req.headers[headerName]) ? req.headers[headerName][0] : req.headers[headerName];` then constant-time compare to the secret.

### HIGH · general

- **Location:** `packages/app-store/_utils/oauth/parseRefreshTokenResponse.ts:1–32`
- **Lens:** llm
- **Rationale:** The Zod schema is invalid/ineffective: using computed keys like `[z.string().toString()]` does not create a catchall and will not validate dynamic properties as intended. Also `parseRefreshTokenResponse` returns the entire safeParse result object rather than `data`, but call sites treat it like credential data, likely breaking runtime behavior and Prisma updates.
- **Suggestion:** Replace with `z.object({ access_token: z.string() }).catchall(z.union([z.number(), z.unknown()]))` or explicitly include `expiry_date` when needed. Return `refreshTokenResponse.data` (not the parse result). Update call sites accordingly and add unit tests.

### HIGH · authz

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:81–110`
- **Lens:** llm
- **Rationale:** After refresh, code sets `googleCredentials.access_token/expiry_date` from `res.data`, but when sync endpoint is used, the response shape/content-type is unknown and may not match `res.data`. Additionally `parseRefreshTokenResponse(googleCredentials, googleCredentialSchema)` (as implemented) returns a parse result object and not the validated credential payload, potentially storing malformed data in Prisma.
- **Suggestion:** Define and enforce a strict response contract for the sync endpoint (e.g., JSON with `access_token` and `expiry_date`). Parse `await res.json()` (and handle non-2xx) before assigning into credentials. Fix `parseRefreshTokenResponse` to return validated payload and ensure `googleCredentialSchema` is applied correctly.

### MEDIUM · security

- **Location:** `packages/app-store/_utils/oauth/refreshOAuthTokens.ts:1–22`
- **Lens:** llm
- **Rationale:** The sync fetch does not set `Content-Type`, does not check `response.ok`, and does not send any authentication to the sync endpoint. If CALCOM_CREDENTIAL_SYNC_ENDPOINT is misconfigured or intercepted, token refresh could silently fail or leak metadata. Using URLSearchParams without headers may be inconsistently handled by the server.
- **Suggestion:** Add headers (`Content-Type: application/x-www-form-urlencoded`), enforce HTTPS, include an auth header (shared secret or mTLS), and handle non-2xx responses with explicit errors. Add timeout/retry/backoff behavior if needed.

## Merged duplicates

### HIGH · general (duplicate)

- **Location:** `apps/web/pages/api/webhook/app-credential.ts:20–40`
- **Lens:** llm
- **Rationale:** Header lookup is brittle: Node/Next lowercases header keys, and `process.env.CALCOM_WEBHOOK_HEADER_NAME` may be configured with uppercase/mixed case. Also, `req.headers[...]` can be `string | string[] | undefined`, so strict comparison to a string env var may fail unexpectedly.
- **Merged into:** `llm.app-credential.ts`

### HIGH · security (duplicate)

- **Location:** `apps/web/pages/api/webhook/app-credential.ts:55–63`
- **Lens:** llm
- **Rationale:** Decryption uses `process.env.CALCOM_APP_CREDENTIAL_ENCRYPTION_KEY || ''`. If the env var is missing/misconfigured, decryption will run with an empty key, which can cause predictable failures or (depending on implementation) insecure behavior. JSON.parse on decrypted content can also throw and return a 500 without a controlled error response.
- **Merged into:** `llm.app-credential.ts`

### MEDIUM · general (duplicate)

- **Location:** `apps/web/pages/api/webhook/app-credential.ts:65–90`
- **Lens:** llm
- **Rationale:** Credential lookup uses `appId: appMetadata.slug`, but Prisma `credential.appId` may represent an internal app id rather than slug (depending on schema). If `appId` is not the slug, this will create duplicate credentials or fail to update the correct record.
- **Merged into:** `llm.app-credential.ts`

### MEDIUM · authz (duplicate)

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:81–110`
- **Lens:** llm
- **Rationale:** In sync mode, the refresh path appears to no longer require/expect a real `refresh_token` and even sets a placeholder elsewhere; this changes assumptions across integrations and could break providers that require refresh_token persistence or validation.
- **Merged into:** `llm.calendarservice.ts`

### MEDIUM · general (duplicate)

- **Location:** `apps/web/pages/api/webhook/app-credential.ts:1–93`
- **Lens:** llm
- **Rationale:** This adds a security-sensitive webhook that writes credentials, but no tests are shown for auth checks, schema validation failures, missing envs, decrypt failures, and upsert behavior.
- **Merged into:** `llm.app-credential.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/app-store/_utils/oauth/parseRefreshTokenResponse.ts:1–32`
- **Lens:** llm
- **Rationale:** Core token-refresh parsing logic is changed/introduced but lacks unit tests; given the schema issues, regressions are likely across multiple OAuth integrations.
- **Merged into:** `llm.parserefreshtokenresponse.ts`
