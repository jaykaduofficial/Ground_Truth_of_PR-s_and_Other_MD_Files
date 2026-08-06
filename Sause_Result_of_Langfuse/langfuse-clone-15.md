# PR Review: jaykaduofficial/langfuse-clone #15

- **PR:** https://github.com/jaykaduofficial/langfuse-clone/pull/15
- **Base scope:** `code:jaykaduofficial/langfuse-clone@main`
- **PR scope:** `code:jaykaduofficial/langfuse-clone@pr:15`
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 3:27:26 PM

## Metrics

- **Findings:** 2 unique (7 raw) · **Files flagged:** 2 · **Density:** 0.3 findings/file
- **Severity:** critical 0 · high 0 · medium 2
- **Files changed:** 6
- **Route:** code_pr_ensemble
- **By category:** security 1 · general 1
- **Top files:** types.ts (1), promptRouter.ts (1)
- **Sources:** lens 0 · llm 7 · merged 2
- **Duplicates merged:** 5

## Summary

1) In `promptRouter.ts` the `protectedLabelCount` calculation (`COUNT(*) FILTER (WHERE cardinality(labels) > 0)`) appears to count any version with labels, not specifically “protected” labels—confirm the intended definition and adjust the query accordingly.  
2) In `types.ts`, `includeArchived` defaults to `true` in the Zod schema, but the UI always passes `includeArchived: false`, creating a mismatch in defaults/behavior; align the default and client usage to avoid surprising results.

## Findings

### MEDIUM · security

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:274–336`
- **Lens:** llm
- **Rationale:** The query calculates `protectedLabelCount` as `COUNT(*) FILTER (WHERE cardinality(labels) > 0)`, which counts any version with any label (including non-protected/custom labels). The UI label says "Labeled versions" but the field is named `protectedLabelCount`, creating a semantic mismatch that will confuse users and future maintainers and may misrepresent the metric.
- **Suggestion:** Either (a) rename the field everywhere to `labeledVersionCount` (schema/type/UI) to match the SQL, or (b) change the SQL filter to count only the intended "protected" labels (e.g., `labels && ARRAY['production','latest']` or whatever is considered protected) and keep the name.

### MEDIUM · general

- **Location:** `packages/shared/src/features/prompts/types.ts:92–114`
- **Lens:** llm
- **Rationale:** `includeArchived` defaults to `true` in the Zod schema, but the UI query always passes `includeArchived: false`. This mismatch can lead to inconsistent behavior across callers and accidental inclusion of archived prompts if a caller omits the field.
- **Suggestion:** Align defaults with expected product behavior (likely default to `false`), or make it required (no default) so callers must choose explicitly.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:310–336`
- **Lens:** llm
- **Rationale:** Bigint-to-Number conversion can overflow if counts become large (JS Number max safe integer). While unlikely for many apps, this is a common footgun with analytics-like aggregations and can silently produce incorrect UI values.
- **Merged into:** `llm.promptrouter.ts`

### MEDIUM · security (duplicate)

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:289–309`
- **Lens:** llm
- **Rationale:** The raw SQL uses `ILIKE %${input.name}%` which can be expensive and potentially leveraged to create heavy queries (DoS-by-query) on large tables, especially since this runs on every list-view load and search change. Although parameterized (so injection risk is low), unbounded wildcard search is a performance/security smell.
- **Merged into:** `llm.promptrouter.ts`

### MEDIUM · general (duplicate)

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:289–309`
- **Lens:** llm
- **Rationale:** The `label` filter checks `${input.label} = ANY(labels)` while `includeArchived` separately excludes `archived`. If a caller provides `label='archived'` with `includeArchived=false`, the conditions conflict and the summary will always return zeros. That may be surprising and produce confusing UI when filters are combined.
- **Merged into:** `llm.promptrouter.ts`

### MEDIUM · general (duplicate)

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:274–336`
- **Lens:** llm
- **Rationale:** A new aggregation endpoint with multiple filters (projectId, name, label, includeArchived) and bigint casting is introduced without tests, increasing risk of regressions and incorrect metrics in the dashboard.
- **Merged into:** `llm.promptrouter.ts`

### INFO · authz (duplicate)

- **Location:** `web/src/features/prompts/server/routers/promptRouter.ts:274–336`
- **Lens:** llm
- **Rationale:** Adds a new `prompts.versionSummary` tRPC procedure and shared types, which is non-breaking but expands the API surface and should be versioned/communicated if clients outside this repo consume the router types.
- **Merged into:** `llm.promptrouter.ts`
