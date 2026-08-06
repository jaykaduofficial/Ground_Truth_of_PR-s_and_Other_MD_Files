# PR Review: jaykaduofficial/saleor-clone #20

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/20
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:20`
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 5:47:45 PM

## Metrics

- **Findings:** 5 unique (9 raw) · **Files flagged:** 4 · **Density:** 0.8 findings/file
- **Severity:** critical 0 · high 0 · medium 4 · info 1
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **By category:** general 2 · data_integrity 1 · security 1 · authz 1
- **Top files:** resolvers.py (2), filters.py (1), schema.py (1), test_gift_cards.py (1)
- **Sources:** lens 1 · llm 8 · merged 5
- **Duplicates merged:** 4

## Summary

The PR adds a new `gift_card_summary` GraphQL query, but there are a few medium-risk issues: a nullable field is used without a guard in a resolver/model method, the `tagged_count=Count("tags")` aggregation may yield misleading/inflated counts, and tag filtering uses `icontains` which can match unintended tag names. Tests cover basic staff/app behavior and threshold override but miss key edge cases (e.g., multi-currency data), and there’s an informational note about `required=True` on the new schema field.

## Findings

### MEDIUM · data_integrity

- **Location:** `saleor/graphql/giftcard/resolvers.py:29`
- **Lens:** data_integrity
- **Rationale:** Nullable field used without guard in model method.
- **Suggestion:** Add None checks before dereferencing.
- **Evidence:** def resolve_gift_card_summary(info, filter=None):

### MEDIUM · security

- **Location:** `saleor/graphql/giftcard/resolvers.py:27–96`
- **Lens:** llm
- **Rationale:** `tagged_count=Count("tags")` inside an aggregate can produce misleading results (and potentially inflated counts) due to joins and duplication, especially when gift cards have multiple tags and other aggregates are computed in the same query. If the intent is “number of gift cards that have at least one tag”, `Count("tags")` is not correct; if the intent is “number of tag relations”, it should be explicit and use `distinct` where appropriate.
- **Suggestion:** Clarify semantics and adjust aggregation: for 'gift cards with tags' use `Count("id", filter=Q(tags__isnull=False), distinct=True)`; for 'total tag assignments' use `Count("tags", distinct=True)` or `Count("tags")` on a separate query to avoid join side-effects.

### MEDIUM · general

- **Location:** `saleor/graphql/giftcard/filters.py:134–170`
- **Lens:** llm
- **Rationale:** Tag filtering uses `tags__name__icontains=tag`, which does substring matching and may match unintended tags (e.g., 'vip' matches 'nonvip'). Also, it can introduce duplicates without `.distinct()` if a gift card has multiple matching tags, affecting aggregates/counts.
- **Suggestion:** Consider exact/iexact match if the API expects a tag name, and add `qs = qs.filter(...).distinct()` for tag filtering to avoid duplication in counts/sums.

### MEDIUM · general

- **Location:** `saleor/graphql/giftcard/tests/queries/test_gift_cards.py:1–304`
- **Lens:** llm
- **Rationale:** Tests cover staff/app basic summary and threshold override, but important edge cases are not covered: multi-currency datasets, tag filtering correctness (and avoiding duplicates), gift cards with multiple tags, expired vs inactive overlap, and behavior when no currency exists (empty queryset).
- **Suggestion:** Add tests for: (1) mixed currencies without filter (should error or return null Money), (2) tag filter returns expected counts and does not inflate totals, (3) gift card with multiple tags does not break/overcount, (4) expired card counting behavior, (5) empty result set returns consistent currency/Money fields.

### INFO · authz

- **Location:** `saleor/graphql/giftcard/schema.py:70–120`
- **Lens:** llm
- **Rationale:** A new query field `gift_card_summary` is added with `required=True` on the field itself, which is fine for additions but may change client expectations if introspection/typing assumes optional query fields (usually not an issue). The filter input is marked as added in 3.22 and permission-protected.
- **Suggestion:** Ensure `GiftCardSummary` type is included in the schema changelog/docs and that the field nullability matches intended behavior (consider whether the resolver can return `None` when currency is `None`/empty set).

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/giftcard/resolvers.py:27–96`
- **Lens:** llm
- **Rationale:** `currency` is derived from `_get_summary_currency(qs, filter)` which, when no currency filter is provided, returns the first currency found while totals are aggregated across all currencies. This can label multi-currency totals with an arbitrary single currency, producing incorrect Money values.
- **Merged into:** `llm.resolvers.py`

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/giftcard/resolvers.py:27–96`
- **Lens:** llm
- **Rationale:** `expired_count=Count("id", filter=Q(expiry_date__lte=today))` counts expired gift cards regardless of `is_active`. Meanwhile `active_count` excludes expired ones. Depending on business meaning, `inactive_count` and `expired_count` may overlap (expired but inactive), making the summary hard to interpret.
- **Merged into:** `llm.resolvers.py`

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/giftcard/resolvers.py:27–96`
- **Lens:** llm
- **Rationale:** Low balance threshold defaults to `10.00` and uses `current_balance_amount__lte`. This includes negative balances (if possible) and also includes fully redeemed (0) gift cards; that may be intended, but also the filter input is optional and there’s no validation for negative thresholds.
- **Merged into:** `llm.resolvers.py`

### MEDIUM · security (duplicate)

- **Location:** `saleor/graphql/giftcard/resolvers.py:27–96`
- **Lens:** llm
- **Rationale:** While permission checks are enforced at the schema field level, the summary query performs aggregates over the entire gift card set. If there are channel/organization scoping rules in the app, this could unintentionally expose cross-scope totals to users with broad permissions.
- **Merged into:** `llm.resolvers.py`
