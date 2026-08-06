# Golden Comments Evaluation Report

**PR:** feat: Sync app credentials between Cal.com & self-hosted platforms (#1)
**Repo:** lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR11059__20260430

---

## Golden Comment 1

**Comment:** The `parseRefreshTokenResponse` function incorrectly sets `refresh_token` to the hardcoded string `'refresh_token'` when it's missing from the OAuth refresh token response. This invalidates the token, breaking subsequent token refreshes and causing authentication failures.

**Verdict:** Correct

**Reason:** The function does exactly this — when the parsed response lacks a `refresh_token` field, instead of falling back to the existing/previous refresh token or leaving it undefined, it assigns the literal string `"refresh_token"` as the value. Any code later using this field to make a real refresh-token API call would send the literal text "refresh_token" instead of an actual token, which would fail authentication with the OAuth provider on the next refresh cycle.

**Evidence:**
```js
if (!refreshTokenResponse.data.refresh_token) {
  refreshTokenResponse.data.refresh_token = "refresh_token";
}
```
File: `packages/app-store/_utils/oauth/parseRefreshTokenResponse.ts`

**Confidence:** High — the hardcoded string assignment is unambiguous in the diff.

---

## Golden Comment 2

**Comment:** Invalid Zod schema syntax. Computed property keys like `[z.string().toString()]` are not valid in Zod object schemas and will cause runtime errors.

**Verdict:** Correct

**Reason:** `z.string().toString()` does not produce a dynamic/pattern key usable for matching arbitrary properties — it simply calls `.toString()` on the Zod schema instance itself, which yields a fixed string like `"[object Object]"` (or similar non-meaningful string representation), not a wildcard or pattern matcher. Used as a computed property key in an object literal, this just creates one (or two colliding) static keys with that string value, not a schema that "matches any property with a number" or "allows other properties," as the inline comments claim. Zod's `z.object()` does not support dynamic/wildcard keys this way (that would require `z.record()`), so the intended behavior — accepting unknown numeric expiry fields and arbitrary additional fields — is not actually achieved, and the schema is effectively broken with respect to its stated purpose.

**Evidence:**
```js
const minimumTokenResponseSchema = z.object({
  access_token: z.string(),
  // Assume that any property with a number is the expiry
  [z.string().toString()]: z.number(),
  // Allow other properties in the token response
  [z.string().optional().toString()]: z.unknown().optional(),
});
```
File: `packages/app-store/_utils/oauth/parseRefreshTokenResponse.ts`

**Confidence:** High — this is a clear misuse of computed property keys; the intended dynamic-key behavior cannot be achieved this way in a plain object literal regardless of Zod.

---

## Golden Comment 3

**Comment:** `parseRefreshTokenResponse` returns a Zod `safeParse` result (`{ success, data, error }`), not the credential key object. Persisting that as `key` stores the wrapper instead of the token payload; we should store the parsed data or use schema `.parse()`.

**Verdict:** Correct

**Reason:** `parseRefreshTokenResponse` returns `refreshTokenResponse`, which is the output of `schema.safeParse(response)` (or `minimumTokenResponseSchema.safeParse(response)`) — this is always a `{ success, data, error }`-shaped object, never the unwrapped token data directly. In the Google Calendar integration, the return value is assigned directly to `key` and persisted via `prisma.credential.update({ data: { key } })`, without ever accessing `.data` on it. This means the actual stored credential is the safeParse wrapper object, not the token payload, which would break any code that later reads `credential.key.access_token` or similar fields expecting the raw token shape.

**Evidence:**
```js
const key = parseRefreshTokenResponse(googleCredentials, googleCredentialSchema);
await prisma.credential.update({
  where: { id: credential.id },
  data: { key },
});
```
File: `packages/app-store/googlecalendar/lib/CalendarService.ts`

Combined with `parseRefreshTokenResponse`'s definition, which always `return refreshTokenResponse;` (the safeParse result), confirms the wrapper — not `.data` — is what gets persisted.

**Confidence:** High — directly traceable by following the return value of `parseRefreshTokenResponse` into its call site.

---

## Golden Comment 4

**Comment:** When `APP_CREDENTIAL_SHARING_ENABLED` and `CALCOM_CREDENTIAL_SYNC_ENDPOINT` are set, the `refreshOAuthTokens` helper returns the fetch `Response`, but several callers (for example `GoogleCalendarService.refreshAccessToken` expecting `res.data`, and `HubspotCalendarService.refreshAccessToken` expecting a `HubspotToken`) assume it returns the integration-specific token object. That mismatch will cause runtime errors in the sync-enabled path unless the return type or those call sites are adjusted.

**Verdict:** Correct

**Reason:** `refreshOAuthTokens` has two branches: when sync is enabled it `return`s the raw `fetch()` Response from `CALCOM_CREDENTIAL_SYNC_ENDPOINT`; otherwise it returns whatever `refreshFunction()` (the integration-specific logic) returns. Callers were written assuming the non-sync shape. Google's caller does `const res = await refreshOAuthTokens(...); const token = res?.data;` — this is only valid for the legacy googleAuth library's return shape, not a generic `Response`. Hubspot's caller types the result as `HubspotToken` directly from `refreshOAuthTokens(...)`, again assuming the integration-specific shape rather than a possible `Response` object. Since the helper is shared infrastructure with two divergent return shapes depending on env config, and call sites don't branch on which shape they're getting, this is a real type/runtime mismatch in the sync-enabled path.

**Evidence:**
```js
// refreshOAuthTokens.ts
if (APP_CREDENTIAL_SHARING_ENABLED && process.env.CALCOM_CREDENTIAL_SYNC_ENDPOINT && userId) {
  const response = await fetch(process.env.CALCOM_CREDENTIAL_SYNC_ENDPOINT, { ... });
  return response;
} else {
  const response = await refreshFunction();
  return response;
}
```
```js
// GoogleCalendarService.ts
const res = await refreshOAuthTokens(async () => { ...; return fetchTokens.res; }, "google-calendar", credential.userId);
const token = res?.data;
```
```ts
// HubspotCalendarService.ts
const hubspotRefreshToken: HubspotToken = await refreshOAuthTokens(async () => hubspotClient.oauth.tokensApi.createToken(...), "hubspot", credential.userId);
```

**Confidence:** High — the divergent return paths and the unconditional shape assumptions at call sites are both clearly visible in the diff.

---

## Golden Comment 5

**Comment:** When the sync endpoint path is used, `res` is a fetch `Response` and has no `.data`; `res?.data` will be undefined and `token.access_token` will throw at runtime. This relies on a consistent return shape from `refreshOAuthTokens`, which isn't guaranteed currently.

**Verdict:** Correct

**Reason:** This is the concrete Google-specific instance of the broader issue described in Golden Comment 4. A standard `fetch` `Response` object does not have a `.data` property, so `res?.data` evaluates to `undefined` when the sync-enabled branch is taken. The subsequent line `googleCredentials.access_token = token.access_token;` would then throw a `TypeError` (cannot read property of undefined), since `token` is `undefined`.

**Evidence:**
```js
const res = await refreshOAuthTokens(
  async () => {
    const fetchTokens = await myGoogleAuth.refreshToken(googleCredentials.refresh_token);
    return fetchTokens.res;
  },
  "google-calendar",
  credential.userId
);
const token = res?.data;
googleCredentials.access_token = token.access_token;
```
File: `packages/app-store/googlecalendar/lib/CalendarService.ts`

**Confidence:** High — directly traceable; `fetch` Response objects do not expose a `.data` field, so the failure mode follows necessarily from the code as written.

---

## Summary

| Metric | Count |
|---|---|
| Total correct golden comments | 5 |
| Total incorrect / partially correct | 0 |

**Overall quality assessment:** All five golden comments are accurate and well-supported by the diff. They collectively surface a coherent cluster of bugs centered on the new shared OAuth-refresh helpers (`parseRefreshTokenResponse` and `refreshOAuthTokens`): a hardcoded placeholder refresh token, an invalid/no-op dynamic Zod schema, persisting the safeParse wrapper instead of the unwrapped token data, and a return-type mismatch between the sync-enabled and legacy code paths of `refreshOAuthTokens` (with comment 5 providing a concrete, verifiable instance of the general issue raised in comment 4). All five are technically precise, cite the exact problematic lines, and correctly reason about runtime consequences. This is a high-quality, low-noise comment set with no false positives, though comments 4 and 5 have meaningful overlap (general claim vs. one concrete example) and could arguably be consolidated.
