# PR Review: jaykaduofficial/saleor-clone #21

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/21
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:21`
- **Files changed:** 7
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 5:54:46 PM

## Metrics

- **Findings:** 5 unique (6 raw) · **Files flagged:** 4 · **Density:** 0.7 findings/file
- **Severity:** critical 0 · high 1 · medium 3 · info 1
- **Files changed:** 7
- **Route:** code_pr_ensemble
- **By category:** general 3 · data_integrity 1 · authz 1
- **Top files:** resolvers.py (2), filters.py (1), schema.py (1), test_vouchers.py (1)
- **Sources:** lens 1 · llm 5 · merged 5
- **Duplicates merged:** 1

## Summary

The PR adds an additive `voucher_usage_summary` GraphQL field gated by `MANAGE_DISCOUNTS`, but there are correctness and completeness issues. A nullable model field is used without a guard in the resolver, and the `remaining_uses=Sum("usage_limit") - Sum("codes__used")` calculation appears mathematically wrong for some voucher models (with tests currently reinforcing that potentially incorrect business logic). Additionally, the new `include_inactive_codes` filter is declared and tested but not actually wired into the filtering logic.

## Findings

### HIGH · general

- **Location:** `saleor/graphql/discount/filters.py:148–184`
- **Lens:** llm
- **Rationale:** The new `include_inactive_codes` filter is declared in `VoucherUsageSummaryFilterInput` and is tested, but it is never applied in `filter_voucher_usage_summary` or in the resolver aggregates. As implemented, inactive codes will always be included in `code_count`/`used_count`, making the API behavior inconsistent with its schema and intent.
- **Suggestion:** Apply `include_inactive_codes` to the aggregates by filtering the joined `codes` relation (e.g., conditionally add `Q(codes__is_active=True)` to `Count("codes")`/`Sum("codes__used")`, or annotate a filtered queryset/CTE). Update tests to assert the default behavior (inactive excluded) and the opt-in behavior (inactive included).

### MEDIUM · data_integrity

- **Location:** `saleor/graphql/discount/resolvers.py:36`
- **Lens:** data_integrity
- **Rationale:** Nullable field used without guard in model method.
- **Suggestion:** Add None checks before dereferencing.
- **Evidence:** def resolve_voucher_usage_summary(info, channel_slug=None, filter=None):

### MEDIUM · general

- **Location:** `saleor/graphql/discount/resolvers.py:36–78`
- **Lens:** llm
- **Rationale:** `remaining_uses=Sum("usage_limit") - Sum("codes__used")` is mathematically incorrect for voucher models where `usage_limit` is per-voucher (not per-code). Joining to `codes` multiplies voucher rows, so `Sum("usage_limit")` will be inflated by the number of codes per voucher, producing wrong remaining totals (and potentially negative/incorrect values).
- **Suggestion:** Compute remaining uses without join-multiplication: e.g., aggregate `used_count` from codes separately and aggregate `usage_limit` over distinct vouchers (using `Sum("usage_limit", distinct=True)` if supported, or a subquery/annotation per voucher with `used_sum=Sum("codes__used")` and then sum `usage_limit - used_sum` across vouchers). Add a test with multiple codes per limited voucher to validate correctness.

### MEDIUM · general

- **Location:** `saleor/graphql/discount/tests/queries/test_vouchers.py:205–340`
- **Lens:** llm
- **Rationale:** Tests validate the new query but currently encode potentially incorrect business logic (e.g., `remainingUses` expecting inflated totals) and do not assert the default behavior for inactive codes. There is also no test coverage for `started` range filtering or for active vs expired boundary conditions.
- **Suggestion:** Add/adjust tests to (1) verify `includeInactiveCodes` defaults to excluding inactive codes, (2) validate `remainingUses` for vouchers with multiple codes (no multiplication), (3) cover `started` filtering, and (4) cover active/expired classification at boundary times (start_date/end_date).

### INFO · authz

- **Location:** `saleor/graphql/discount/schema.py:138–176`
- **Lens:** llm
- **Rationale:** A new GraphQL field `voucher_usage_summary` is added behind `MANAGE_DISCOUNTS` permission. This is additive and should be non-breaking, but it is a public API surface area increase that should align with versioning and documentation expectations.
- **Suggestion:** Ensure the `VoucherUsageSummary` type definition is present/complete (including correct field names and types) and that documentation/changelog references match `ADDED_IN_322` usage.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/discount/resolvers.py:36–78`
- **Lens:** llm
- **Rationale:** `code_count=Count("codes")` and `used_count=Sum("codes__used")` may be skewed when the base voucher queryset is filtered by `Q(name__icontains=...) | Q(codes__code__icontains=...)`. The OR condition can introduce join duplication and inflate counts/sums unless `distinct=True` or the query is structured to avoid duplicate voucher rows.
- **Merged into:** `llm.resolvers.py`
