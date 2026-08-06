# Golden Comments Evaluation Report

**PR:** AuthZService: improve authz caching (#1)
**Repo:** lyxor-pr-testing-org/grafana__grafana__lyxor__PR103633__20260504

---

## Golden Comment 1

**Comment:** The Check operation exhibits asymmetric cache trust logic: cached permission grants are trusted and returned immediately, but cached denials from the same permission cache are ignored, leading to a fresh database lookup. This allows stale cached grants to provide access to revoked resources, posing a security risk.

**Verdict:** Correct

**Reason:** In `Check`, after the `permDenialCache` lookup, the code calls `getCachedIdentityPermissions` and, if it succeeds (`err == nil`), runs `checkPermission(ctx, cachedPerms, checkReq)`. If `allowed` is `true`, it returns immediately with `Allowed: true` straight from the cache. If `allowed` is `false` (either because the cached permission map has an explicit `false`/absent entry for the requested scope, as in the `dashboards:uid:dash1: false` and the missing-`dash2` test cases), the code does **not** return the cached `false` result — it falls through to a fresh `getIdentityPermissions` database call instead. This confirms the described asymmetry: cache hits are trusted only when they resolve to "allowed," and any non-allow result from the same `permCache` map is treated as inconclusive and re-verified against the database. Since `permCache` entries persist for `shortCacheTTL`, a permission that was true at cache-write time but has since been revoked can still be served as `Allowed: true` until the cache entry expires or is evicted — this is a legitimate, currently-unaddressed staleness/security consideration directly attributable to this code path.

**Evidence:**
```go
// service.go - Check()
cachedPerms, err := s.getCachedIdentityPermissions(ctx, checkReq.Namespace, checkReq.IdentityType, checkReq.UserUID, checkReq.Action)
if err == nil {
    allowed, err := s.checkPermission(ctx, cachedPerms, checkReq)
    if err != nil {
        ...
        return deny, err
    }
    if allowed {
        ...
        return &authzv1.CheckResponse{Allowed: allowed}, nil
    }
}
// falls through to a fresh DB lookup if allowed was false
permissions, err := s.getIdentityPermissions(ctx, checkReq.Namespace, checkReq.IdentityType, checkReq.UserUID, checkReq.Action)
```

**Confidence:** Medium-High — the asymmetric control flow (trust-on-allow, re-verify-on-deny) is unambiguous in the diff and the staleness implication for grants is a sound general security observation. Confidence is not higher only because this is a deliberate (if debatable) caching trade-off rather than an obviously unintended bug, and similar trust-on-cache-hit behavior for `permCache` existed prior to this PR's refactor as well, so the "introduced by this PR" framing is slightly imprecise even though the described mechanism is accurate.

---

## Golden Comment 2

**Comment:** The test comment says the cached permissions 'allow access', but the map stores false for dashboards:uid:dash1, so checkPermission will still treat this scope as not allowed.

**Verdict:** Correct

**Reason:** In the `TestService_CacheCheck` subtest `"Should deny on explicit cache deny entry"`, the inline comment reads `// Allow access to the dashboard to prove this is not checked`, immediately preceding a `permCache.Set` call that stores `map[string]bool{"dashboards:uid:dash1": false}`. A value of `false` for that scope means `checkPermission` will evaluate the dashboard as not allowed, not allowed — the comment's wording directly contradicts the data being set. This is a genuine documentation/comment mismatch in the test code (the intent was presumably to show that even if `permCache` were checked, it would also resolve to deny, or the comment text is simply stale/incorrect), though it doesn't affect the test's actual assertions, which correctly expect `resp.Allowed` to be `false`.

**Evidence:**
```go
// service_test.go - TestService_CacheCheck / "Should deny on explicit cache deny entry"
// Allow access to the dashboard to prove this is not checked
s.permCache.Set(ctx, userPermCacheKey("org-12", "testuid", "dashboards:read"), map[string]bool{"dashboards:uid:dash1": false})
```

**Confidence:** High — the mismatch between the comment text and the literal boolean value being set is directly visible and unambiguous in the diff.

---

## Summary

| Metric | Count |
|---|---|
| Total correct golden comments | 2 |
| Total incorrect / partially correct | 0 |

**Overall quality assessment:** Both golden comments are accurate and well-grounded in the diff. The first correctly identifies a real asymmetry in how `Check` treats cache hits — trusting a cached "allow" immediately while re-verifying anything else against the database — and reasonably flags the resulting staleness window as a security-relevant consideration, though it's worth noting this trust-on-allow pattern isn't entirely new to this PR and reflects an intentional (if debatable) performance/staleness trade-off rather than an obviously unintended regression. The second is a precise, low-ambiguity catch of a test comment that contradicts the actual cached value it describes — a small but legitimate documentation defect that could confuse future readers of the test, even though it doesn't affect test correctness. Both comments cite specific, verifiable lines and accurately describe the underlying behavior.
