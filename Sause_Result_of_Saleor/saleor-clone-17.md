# PR Review: jaykaduofficial/saleor-clone #17

- **PR:** https://github.com/jaykaduofficial/saleor-clone/pull/17
- **Base scope:** `code:jaykaduofficial/saleor-clone@main`
- **PR scope:** `code:jaykaduofficial/saleor-clone@pr:17`
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 5:32:43 PM

## Metrics

- **Findings:** 3 unique (5 raw) · **Files flagged:** 3 · **Density:** 0.8 findings/file
- **Severity:** critical 0 · high 1 · medium 2
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **By category:** general 2 · authz 1
- **Top files:** schema.py (1), schema_printer.py (1), test_utils.py (1)
- **Sources:** lens 0 · llm 5 · merged 3
- **Duplicates merged:** 2

## Summary

Found a high-severity issue: the new test `test_print_schema_can_sort_enum_values` ends with `assert printed_schema.` which is syntactically invalid and will break the test suite. Medium-severity concerns include argument sorting being based on the argument *type* (`str(item[1].type).lower()`) rather than argument name, and federated SDL output now defaulting to `sort_schema=True`, potentially changing SDL text ordering and impacting downstream tools that rely on exact output.

## Findings

### HIGH · general

- **Location:** `saleor/graphql/tests/test_utils.py:86–120`
- **Lens:** llm
- **Rationale:** The new test `test_print_schema_can_sort_enum_values` ends with `assert printed_schema.` which is syntactically invalid and will cause the test suite to fail.
- **Suggestion:** Complete the assertion by comparing the index positions of the enum values in the printed SDL, e.g. `assert printed_schema.index("  ALPHA") < printed_schema.index("  ZED")`, and ensure the test file runs/compiles.

### MEDIUM · general

- **Location:** `saleor/graphql/schema_printer.py:351–374`
- **Lens:** llm
- **Rationale:** Argument sorting uses `key=lambda item: str(item[1].type).lower()` (type-based sorting) rather than sorting by argument name. This can reorder args in a surprising, non-intuitive way and may not be stable when multiple args share the same type (tie order depends on insertion order), undermining the goal of deterministic output across sources.
- **Suggestion:** Sort args by argument name (`item[0].lower()`) or use a stable composite key like `(item[0].lower(), str(item[1].type).lower())` depending on desired semantics.

### MEDIUM · authz

- **Location:** `saleor/graphql/core/federation/schema.py:173`
- **Lens:** llm
- **Rationale:** Federated SDL output changes ordering by default (`sort_schema=True`). If downstream tooling depends on exact SDL text ordering (e.g., snapshot tests, schema registry diffing configured to be order-sensitive, or custom parsers), this can be a behavior change.
- **Suggestion:** Document the ordering change in release notes/changelog and consider making sorting configurable (env/setting) for federation as well if consumers rely on the previous order.

## Merged duplicates

### INFO · general (duplicate)

- **Location:** `saleor/graphql/schema_printer.py:63–147`
- **Lens:** llm
- **Rationale:** Sorting is added for directives, types, fields, input fields, enum values, union members, and args, but tests currently only cover type + field sorting and (intended) enum sorting; there are no tests for sorting of input object fields, union member ordering, directive ordering, or argument ordering (especially multiline args with descriptions).
- **Merged into:** `llm.schema_printer.py`

### INFO · general (duplicate)

- **Location:** `saleor/graphql/schema_printer.py:70–146`
- **Lens:** llm
- **Rationale:** The printer now converts iterators to lists and conditionally sorts; this is fine, but the `type_ = type_` assignments are redundant and the repeated `lower()` key lambdas could be centralized for readability.
- **Merged into:** `llm.schema_printer.py`
