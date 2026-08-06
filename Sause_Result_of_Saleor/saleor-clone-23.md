# PR Review: jaykaduofficial/saleor-clone #23

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/23
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:23`
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **Reviewed:** 7/22/2026, 9:21:28 AM

## Metrics

- **Findings:** 5 unique (7 raw) · **Files flagged:** 4 · **Density:** 1 findings/file
- **Severity:** critical 0 · high 1 · medium 3 · info 1
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **By category:** data_integrity 2 · general 2 · authz 1
- **Top files:** resolvers.py (2), schema.graphql (1), schema.py (1), test_shipping_zones.py (1)
- **Sources:** lens 2 · llm 5 · merged 5
- **Duplicates merged:** 2

## Summary

The new shipping zone summary resolver likely overcounts because it joins across shipping methods/channel listings/channels and aggregates without `distinct=True`, causing row multiplication (high). There are also multiple spots where nullable fields are used without guards in schema/resolvers (medium), and current tests don’t cover scenarios that would expose the join-multiplication bug. The schema change is additive and should be backwards compatible, though the Python schema definition should be kept in sync with the SDL.

## Findings

### HIGH · general

- **Location:** `saleor/graphql/shipping/resolvers.py:22–45`
- **Lens:** llm
- **Rationale:** The resolver uses joins (shipping_methods, channel_listings, channels) and then aggregates counts without `distinct=True`, which can inflate `total_count`, `method_count`, and `channel_count` due to row multiplication when a zone has multiple methods/channel listings/channels. This makes the summary unreliable and can regress dashboards/monitoring that depends on accurate counts.
- **Suggestion:** Use distinct counts for related fields and for the base count when joins are present, e.g. `Count('id', distinct=True)`, `Count('shipping_methods', distinct=True)`, `Count('channels', distinct=True)`. Alternatively compute each count on a fresh queryset (or use Subquery/annotate) to avoid join multiplication.

### MEDIUM · data_integrity

- **Location:** `saleor/graphql/shipping/resolvers.py:20`
- **Lens:** data_integrity
- **Rationale:** Nullable field used without guard in model method.
- **Suggestion:** Add None checks before dereferencing.
- **Evidence:** def resolve_shipping_zone_summary(info, channel_slug=None):

### MEDIUM · data_integrity

- **Location:** `saleor/graphql/shipping/schema.py:94`
- **Lens:** data_integrity
- **Rationale:** Nullable field used without guard in model method.
- **Suggestion:** Add None checks before dereferencing.
- **Evidence:** def resolve_shipping_zone_summary(_root, info: ResolveInfo, *, channel=None):

### MEDIUM · general

- **Location:** `saleor/graphql/shipping/tests/queries/test_shipping_zones.py:1–185`
- **Lens:** llm
- **Rationale:** Tests cover permission denial and a basic single-zone case, but they do not catch the likely join-multiplication bug in aggregates (e.g., a zone with multiple shipping methods and multiple channel listings). Without such a test, inflated counts could ship unnoticed.
- **Suggestion:** Add tests with (1) one zone having 2+ shipping methods and those methods having multiple channel listings, and assert `totalCount==1`, `methodCount==<distinct methods>`, `channelCount==<distinct channels>`. Also add a test for zones assigned to a channel but with no shipping methods (if that should be included).

### INFO · authz

- **Location:** `saleor/graphql/schema.graphql:308–340`
- **Lens:** llm
- **Rationale:** This is an additive schema change (new query and type) and should be backwards compatible; however the Python schema defines the field as `required=True` while also having an optional argument, which can be confusing in terms of client expectations around nullability and permission errors.
- **Suggestion:** Confirm the intended nullability/behavior: if permission is missing, ensure it returns a proper GraphQL error rather than null data. Consider aligning documentation and implementation naming (`shippingZoneSummary` vs `shipping_zone_summary`) and double-check `required=True` is appropriate for a query field.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/shipping/resolvers.py:22–45`
- **Lens:** llm
- **Rationale:** The channel filter scopes zones by `shipping_methods__channel_listings__channel__slug`, which excludes zones that are assigned to the channel via `ShippingZone.channels` but have zero shipping methods or missing channel listings. This can undercount zones and channel assignments for a channel-scoped summary.
- **Merged into:** `llm.resolvers.py`

### MEDIUM · general (duplicate)

- **Location:** `saleor/graphql/shipping/resolvers.py:22–45`
- **Lens:** llm
- **Rationale:** `country_count = sum(len(zone.countries) for zone in qs)` evaluates the queryset in Python, potentially loading many rows and doing O(n) work per request. With many zones this can become slow; additionally, if `qs` has duplicates due to joins (especially when `channel_slug` is provided), you may double-count countries.
- **Merged into:** `llm.resolvers.py`
