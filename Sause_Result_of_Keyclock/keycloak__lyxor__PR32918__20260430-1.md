# PR Review: lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR32918__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR32918__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR32918__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR32918__20260430@pr:1`
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **Reviewed:** 7/14/2026, 2:11:13 PM

## Metrics

- **Findings:** 4 unique (6 raw) · **Files flagged:** 4 · **Density:** 1 findings/file
- **Severity:** critical 0 · high 2 · medium 1 · info 1
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **By category:** general 3 · authz 1
- **Top files:** InfinispanIdentityProviderStorageProvider.java (1), IdentityProviderStorageProvider.java (1), OrganizationAwareIdentityProviderBean.java (1), OrganizationCacheTest.java (1)
- **Sources:** lens 0 · llm 6 · merged 4
- **Duplicates merged:** 2

## Summary

Two high-risk issues: the new `getForLogin()` cache key in `InfinispanIdentityProviderStorageProvider` ignores `organizationId` (only realm+mode), which can cause cross-organization cache pollution, and `LoginFilter.getLoginPredicate()` semantics change such that organization-linked IDPs are excluded unless `BROKER_PUBLIC` is set (potential behavior regression). Medium/info notes: the added `OrganizationCacheTest` snippet is truncated but appears to mix enabled/disabled IDPs (verify expectations/coverage), and the UI re-check of `idp.isEnabled()` suggests wrapper behavior may alter enabled-state perception, indicating a potential layering/leak concern.

## Findings

### HIGH · general

- **Location:** `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/idp/InfinispanIdentityProviderStorageProvider.java:198–271`
- **Lens:** llm
- **Rationale:** The cache key for getForLogin() ignores organizationId (it is only realm+mode). The implementation attempts to store per-organization results inside a single IdentityProviderListQuery keyed by `searchKey`, but the backing cache entry is still a single key; concurrent calls for different organizations can invalidate each other (via cache.invalidateObject(cacheKey)) and cause cache thrash, and any bug in IdentityProviderListQuery's internal map handling could lead to returning wrong org-specific results across requests because organizationId is not part of the cache key.
- **Suggestion:** Include organizationId (or an explicit discriminator for null vs non-null) in the cache key (e.g., realmId + ".idp.login." + mode + ".org." + orgIdOrEmpty) so each organization’s result set is isolated. If you intentionally want a multi-search cache entry, avoid invalidating the whole key when a new searchKey is encountered; instead, mutate/replace the query without invalidating the entire cache entry, and add stress tests for concurrent org lookups.

### HIGH · authz

- **Location:** `server-spi/src/main/java/org/keycloak/models/IdentityProviderStorageProvider.java:251–258`
- **Lens:** llm
- **Rationale:** LoginFilter.getLoginPredicate() semantics change: organization-linked IDPs are now excluded unless BROKER_PUBLIC config is true. This affects all callers of getLoginPredicate()/getForLogin and can change which IDPs appear on login, potentially breaking deployments that relied on org-linked IDPs being visible without explicitly setting BROKER_PUBLIC.
- **Suggestion:** Document this behavioral change in release notes / migration notes and consider backward-compatible behavior (e.g., defaulting BROKER_PUBLIC to true for existing org-linked IDPs, or gating the new predicate logic behind a feature flag). Add tests covering legacy behavior expectations if applicable.

### MEDIUM · general

- **Location:** `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/organization/cache/OrganizationCacheTest.java:365–536`
- **Lens:** llm
- **Rationale:** A new test is added but the diff snippet is truncated; from the shown setup it creates IDPs with mixed enabled/disabled and BROKER_PUBLIC for half. The critical missing coverage is for correctness across different organizationId values (ensuring org-specific results are not mixed), update scenarios that should/shouldn’t invalidate, and concurrent lookups (cache thrash/race) given the shared cache key.
- **Suggestion:** Extend the test to: (1) call getForLogin for two different organizations and assert distinct result sets and stable cache entries; (2) update an IDP property that affects login selection while preserving predicate truth and assert invalidation; (3) add a concurrency test (or at least sequential interleaving) demonstrating that caching does not return wrong results after alternating orgId calls.

### INFO · general

- **Location:** `services/src/main/java/org/keycloak/organization/forms/login/freemarker/model/OrganizationAwareIdentityProviderBean.java:72–85`
- **Lens:** llm
- **Rationale:** Re-checking idp.isEnabled() in the UI layer suggests wrapper behavior can change enabled state perception, which is a leaky abstraction and can mask upstream caching/wrapping issues. While harmless, it indicates the caching/wrapping may be returning models that don't consistently reflect enabled status at filter time.
- **Suggestion:** Prefer ensuring the wrapped model used by getForLogin() preserves isEnabled semantics (or ensure getForLogin() never returns disabled providers) so callers don’t need to defensively re-filter. Add a targeted test to verify disabled IDPs never appear in getForLogin() results even when cached.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/idp/InfinispanIdentityProviderStorageProvider.java:92–111`
- **Lens:** llm
- **Rationale:** remove(alias) now eagerly calls idpDelegate.getByAlias(alias) before knowing whether the IDP exists and before removal. If alias does not exist, storedIdp is null; registerIDPLoginInvalidation(storedIdp) will NPE because getLoginPredicate() starts with Objects::nonNull but is invoked via test(idp) only if you call it—here you call getLoginPredicate().test(idp) inside registerIDPLoginInvalidation, so it is safe only if that method handles null. It currently does, but remove() still does extra DB work and may change behavior (extra read) on removal paths.
- **Merged into:** `llm.infinispanidentityproviderstorageprovider.java`

### MEDIUM · general (duplicate)

- **Location:** `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/idp/InfinispanIdentityProviderStorageProvider.java:323–377`
- **Lens:** llm
- **Rationale:** registerIDPLoginInvalidationOnUpdate() skips invalidation if both original and updated satisfy getLoginPredicate() and organizationId is unchanged. However, getForLogin() results can also depend on other fields (enabled, linkOnly, hideOnLoginPage, firstBrokerLoginFlow/alias, config used by LoginFilters, etc.). If any of those change while still satisfying the predicate, cached getForLogin results may become stale.
- **Merged into:** `llm.infinispanidentityproviderstorageprovider.java`
