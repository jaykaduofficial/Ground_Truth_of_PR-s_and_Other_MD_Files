# PR Review: jaykaduofficial/langfuse-clone #31

- **PR:** https://github.com/jaykaduofficial/langfuse-clone/pull/31
- **Base scope:** `code:jaykaduofficial/langfuse-clone@main`
- **PR scope:** `code:jaykaduofficial/langfuse-clone@pr:31`
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 3:56:13 PM

## Metrics

- **Findings:** 3 unique (5 raw) · **Files flagged:** 3 · **Density:** 0.6 findings/file
- **Severity:** critical 0 · high 0 · medium 2 · info 1
- **Files changed:** 5
- **Route:** code_pr_ensemble
- **By category:** general 3
- **Top files:** PricePreview.tsx (1), PricingSection.tsx (1), utils.ts (1)
- **Sources:** lens 0 · llm 5 · merged 3
- **Duplicates merged:** 2

## Summary

`estimateCostFromPrices` currently sums `price * usageCount` for every `prices` entry, which can overcount if a pricing config includes extra/irrelevant buckets. In `PricePreview`, `Number(event.target.value)` turns an empty cleared input into `0`, unexpectedly resetting usage and affecting the estimate. Also, `PricingSection` hardcodes the “1,000/1,000/2,000” sample-usage copy, so it can drift from `DEFAULT_SAMPLE_USAGE` if that constant changes.

## Findings

### MEDIUM · general

- **Location:** `web/src/features/models/utils.ts:21–52`
- **Lens:** llm
- **Rationale:** estimateCostFromPrices currently sums (price * usageCount) for every entry in `prices`. If a pricing config includes both `total` and `input`/`output` (or other overlapping usage types), the estimate can double-count the same work and produce misleading totals. This is especially likely because the sample usage includes all three dimensions (input/output/total).
- **Suggestion:** Define an explicit precedence/combination rule (e.g., if `total` price exists, ignore input/output; or compute total = input+output and only use one dimension). Alternatively, only sum across a known non-overlapping set of usage types, and add a unit test covering mixed {input, output, total} pricing.

### MEDIUM · general

- **Location:** `web/src/features/models/components/PricePreview.tsx:16–46`
- **Lens:** llm
- **Rationale:** updateSampleUsage casts input values via Number(event.target.value). When the user clears the field, this becomes 0 (empty string -> 0) which may be surprising; other invalid states can produce NaN, which then flows into normalizeSampleUsage and becomes NaN -> Math.round(NaN) -> NaN, potentially propagating and resulting in a NaN cost.
- **Suggestion:** Handle empty/invalid values explicitly: parse with `const v = event.target.value === "" ? undefined : Number(event.target.value)` and let normalization default, or clamp NaN to defaults in normalizeSampleUsage (e.g., `Number.isFinite(x) ? ... : default`). Add tests for empty string and NaN handling.

### INFO · general

- **Location:** `web/src/features/models/components/pricing-tiers/PricingSection.tsx:25–73`
- **Lens:** llm
- **Rationale:** The sample estimate text hardcodes the sample usage (1,000/1,000/2,000). If DEFAULT_SAMPLE_USAGE changes, the UI copy can drift from actual behavior.
- **Suggestion:** Derive the displayed sample usage values from DEFAULT_SAMPLE_USAGE to keep the copy consistent (e.g., interpolate constants) or centralize the copy generation.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `web/src/features/models/utils.ts:54–74`
- **Lens:** llm
- **Rationale:** formatEstimatedCost converts Decimal to a JS number to decide formatting thresholds. For very large values, `toNumber()` can overflow to Infinity or lose precision, and for very small values it may underflow, leading to incorrect formatting decisions even though Decimal itself is precise.
- **Merged into:** `llm.utils.ts`

### INFO · general (duplicate)

- **Location:** `web/src/features/models/utils.ts:1–74`
- **Lens:** llm
- **Rationale:** New core logic (normalizeSampleUsage, estimateCostFromPrices, formatEstimatedCost) is added without accompanying tests, increasing regression risk and making edge cases (mixed usage keys, NaN/undefined usage, formatting thresholds) easy to miss.
- **Merged into:** `llm.utils.ts`
