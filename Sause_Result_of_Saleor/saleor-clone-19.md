# PR Review: jaykaduofficial/saleor-clone #19

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/19
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:19`
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 5:40:19 PM

## Metrics

- **Findings:** 7 · **Files flagged:** 6 · **Density:** 1.2 findings/file
- **Severity:** critical 0 · high 0 · medium 6 · info 1
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **By category:** general 4 · data_integrity 1 · security 1 · authz 1
- **Top files:** resolvers.py (2), filters.py (1), schema.py (1), test_order.py (1), types.py (1)
- **Sources:** lens 1 · llm 6 · merged 7

## Summary

The PR introduces a new order lookup by customer email (including `icontains` fragment matching), which risks customer data disclosure and unintended matches across `user_email`/`user__email`, and current tests don’t cover key authorization/permission and channel-scoping denial cases. There are also correctness issues: nullable email usage without guards, `customer_email_domain` mishandling malformed emails, and “latest” ordering relying only on `created_at` (non-deterministic when timestamps collide). Overall it’s an additive API change but increases the exposed surface area and should be tightened with stricter filtering/permissions and more robust validation/deterministic ordering.

## Findings

### MEDIUM · data_integrity

- **Location:** `saleor/graphql/order/resolvers.py:138`
- **Lens:** data_integrity
- **Rationale:** Nullable field used without guard in model method.
- **Suggestion:** Add None checks before dereferencing.
- **Evidence:** def resolve_latest_order_by_customer_email(info, email, channel_slug=None):

### MEDIUM · security

- **Location:** `saleor/graphql/order/schema.py:194–223`
- **Lens:** llm
- **Rationale:** The new query allows looking up orders by email (including fragments via icontains), which can enable customer data discovery/enumeration if exposed to broader roles or if permission checks are not strictly enforced at the field level.
- **Suggestion:** Ensure the field is protected by an explicit permission/authorization check consistent with other order lookup queries (e.g., require manage_orders or equivalent), and consider restricting matching to exact email only (or minimum fragment length + rate limiting) to reduce enumeration risk.

### MEDIUM · general

- **Location:** `saleor/graphql/order/filters.py:173–180`
- **Lens:** llm
- **Rationale:** filter_customer_email uses icontains across both user_email and user__email, which can return unintended matches (e.g., searching "com" matches many orders) and may lead to surprising results or heavy queries on large datasets.
- **Suggestion:** Consider switching to iexact for full email inputs and only using icontains when explicitly requested; alternatively enforce a minimum query length and/or use istartswith on domain/username parts.

### MEDIUM · general

- **Location:** `saleor/graphql/order/resolvers.py:136–149`
- **Lens:** llm
- **Rationale:** The resolver orders by created_at only; if multiple orders share the same created_at timestamp, the 'latest' result can be non-deterministic across database backends.
- **Suggestion:** Add a deterministic tiebreaker (e.g., order_by('-created_at', '-pk') or '-id') to guarantee stable selection.

### MEDIUM · general

- **Location:** `saleor/graphql/order/types.py:2520–2532`
- **Lens:** llm
- **Rationale:** customer_email_domain uses email.split('@')[-1], which will return the full string if no '@' is present (malformed email), potentially exposing unexpected values and making the field semantics inconsistent.
- **Suggestion:** Validate parsing more strictly (e.g., return None unless email contains exactly one '@' and a non-empty domain) or use a robust email parsing/validation utility.

### MEDIUM · general

- **Location:** `saleor/graphql/order/tests/queries/test_order.py:1597–1672`
- **Lens:** llm
- **Rationale:** Tests cover exact and fragment matching but do not cover authorization/permission denial, channel scoping correctness when multiple channels have orders for the same email, or the case where user_email is null and user__email should match.
- **Suggestion:** Add tests for: (1) missing/insufficient permissions returns an error/null, (2) two channels with same email returns latest only within specified channel, (3) order with user set and user_email empty still matches via user__email.

### INFO · authz

- **Location:** `saleor/graphql/schema.graphql:921–943`
- **Lens:** llm
- **Rationale:** This is an additive API change (new query and new field) and should be safe for existing clients, but it increases the surface area of order lookup capabilities.
- **Suggestion:** Document the intended access control and recommended usage (exact email vs fragment) in the API docs/changelog, and consider marking fragment behavior explicitly to avoid misuse.
