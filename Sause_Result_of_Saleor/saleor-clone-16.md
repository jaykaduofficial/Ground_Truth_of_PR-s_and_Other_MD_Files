# PR Review: jaykaduofficial/saleor-clone #16

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/16
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:16`
- **Files changed:** 11
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 5:28:56 PM

## Metrics

- **Findings:** 4 · **Files flagged:** 4 · **Density:** 0.4 findings/file
- **Severity:** critical 0 · high 1 · medium 2 · info 1
- **Files changed:** 11
- **Route:** code_pr_ensemble
- **By category:** general 3 · authz 1
- **Top files:** tasks.py (1), export.py (1), filters.py (1), sorters.py (1)
- **Sources:** lens 0 · llm 4 · merged 4

## Summary

The `filter_expires_at` implementation is incorrect: it applies the range filter to `updated_at` instead of `expires_at`, breaking GraphQL `expiresAt` filtering. The cleanup task now deletes any expired `ExportFile` regardless of existing associations, and adding `EXPIRES_AT` to the sort enum introduces an (additive) GraphQL schema change. Also, file size calculation relies on `seek/tell`, which may not work for non-seekable file-like objects.

## Findings

### HIGH · general

- **Location:** `saleor/graphql/csv/filters.py:1–40`
- **Lens:** llm
- **Rationale:** `filter_expires_at` applies the range filter to `updated_at` instead of `expires_at`, so GraphQL filtering by `expiresAt` will return incorrect results and can silently mislead clients relying on expiry-based queries.
- **Suggestion:** Change `filter_range_field(qs, "updated_at", value)` to `filter_range_field(qs, "expires_at", value)` and add/adjust a test that would fail if the field is wrong (e.g., set `updated_at` and `expires_at` to divergent values and assert filtering uses `expires_at`).

### MEDIUM · general

- **Location:** `saleor/csv/tasks.py:111–140`
- **Lens:** llm
- **Rationale:** The deletion query now removes any `ExportFile` with `expires_at__lte=now` regardless of whether it still has associated recent events, which may differ from previous retention semantics based on events. If `expires_at` is set incorrectly (clock skew, bad config), active exports could be deleted prematurely.
- **Suggestion:** Confirm intended precedence rules (explicit `expires_at` always wins). If not intended, incorporate event-based retention into the `expires_at` branch (e.g., only delete if expired AND finished/has cleanup event) and add a regression test for an export with `expires_at` in the past but a recent event.

### MEDIUM · authz

- **Location:** `saleor/graphql/csv/sorters.py:1–30`
- **Lens:** llm
- **Rationale:** Adding `EXPIRES_AT` to the `ExportFileSortField` enum changes the GraphQL schema. While additive, it can still be considered a public API change requiring versioning/announcement for downstream clients that pin schema snapshots.
- **Suggestion:** Document the new sort field in the API changelog/release notes and ensure schema snapshot tests (if used) are updated.

### INFO · general

- **Location:** `saleor/csv/utils/export.py:275–285`
- **Lens:** llm
- **Rationale:** File size is computed via `seek(0, 2)` + `tell()`, which assumes the file object supports random access and that `tell()` reflects byte size. Some storage/file wrappers may not behave as expected.
- **Suggestion:** Consider a fallback when `seek/tell` is unsupported (e.g., try/except and omit `file_size`), and add a test with a non-seekable file-like object if such inputs are possible in production.
