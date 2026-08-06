# PR Review: jaykaduofficial/saleor-clone #22

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/22
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:22`
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **Reviewed:** 7/22/2026, 9:16:22 AM

## Metrics

- **Findings:** 3 · **Files flagged:** 3 · **Density:** 0.6 findings/file
- **Severity:** critical 0 · high 0 · medium 2 · info 1
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **By category:** general 2 · authz 1
- **Top files:** schema.graphql (1), resolvers.py (1), test_stocks.py (1)
- **Sources:** lens 0 · llm 3 · merged 3

## Summary

The new stock summary resolver uses `low_stock_threshold` directly in a `quantity__lte` filter without validation, allowing negative values. Tests cover permissions and a single-stock happy path but don’t verify aggregation correctness across multiple stocks. The schema change is additive and non-breaking, but it expands the public API surface.

## Findings

### MEDIUM · general

- **Location:** `saleor/graphql/warehouse/resolvers.py:17–41`
- **Lens:** llm
- **Rationale:** The resolver accepts `low_stock_threshold` directly and uses it in a `quantity__lte` filter without validation. Negative values or very large values can produce surprising results (e.g., negative threshold makes lowStockCount almost always 0; extremely large values makes it equal totalCount). Also, `total_available` can become negative if `quantity_allocated` exceeds `quantity`, which may be possible in edge cases and could confuse clients expecting a non-negative available quantity.
- **Suggestion:** Validate/clamp `low_stock_threshold` (e.g., enforce >= 0 and optionally a reasonable max) and decide on a contract for `total_available` (either allow negatives explicitly and document it, or clamp at 0). Add tests covering negative threshold and allocated > quantity behavior.

### MEDIUM · general

- **Location:** `saleor/graphql/warehouse/tests/queries/test_stocks.py:26–98`
- **Lens:** llm
- **Rationale:** The added tests cover permissions and a single-stock happy path, but do not verify aggregation correctness across multiple stock rows (summing, multiple low-stock matches) or default behavior when the variable is omitted (default threshold=5). This leaves risk that aggregation and default argument wiring could regress.
- **Suggestion:** Add tests with multiple Stock records to assert `totalQuantity`, `totalAllocated`, `totalAvailable`, and `lowStockCount` are correct across rows; add a test calling `stockSummary` without `lowStockThreshold` to confirm the default of 5 is applied.

### INFO · authz

- **Location:** `saleor/graphql/schema.graphql:244–270`
- **Lens:** llm
- **Rationale:** This introduces a new query and type, which is additive and generally non-breaking, but it does expand the API surface area and requires proper permission enforcement.
- **Suggestion:** Ensure the new field is included in any API changelog/release notes and confirm permission enforcement is consistent with other stock-related queries.
