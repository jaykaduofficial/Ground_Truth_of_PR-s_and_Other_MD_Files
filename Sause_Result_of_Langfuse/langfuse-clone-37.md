# PR Review: jaykaduofficial/langfuse-clone #37

- **PR:** https://github.com/jaykaduofficial/langfuse-clone/pull/37
- **Base scope:** `code:jaykaduofficial/langfuse-clone@main`
- **PR scope:** `code:jaykaduofficial/langfuse-clone@pr:37`
- **Files changed:** 7
- **Route:** code_pr_ensemble
- **Reviewed:** 7/21/2026, 4:42:29 PM

## Metrics

- **Findings:** 3 unique (7 raw) · **Files flagged:** 3 · **Density:** 0.4 findings/file
- **Severity:** critical 0 · high 0 · medium 2 · info 1
- **Files changed:** 7
- **Route:** code_pr_ensemble
- **By category:** security 1 · authz 1 · general 1
- **Top files:** duplicate-folder.tsx (1), duplicate-prompt.tsx (1), README.md (1)
- **Sources:** lens 0 · llm 7 · merged 3
- **Duplicates merged:** 3

## Summary

React Hook Form `defaultValues` in `duplicate-folder.tsx` only apply on first render, so the initial `suggestedTargetPath` may be `undefined` and not update as expected. `DuplicatePromptForm` now requires a new `allPromptNames` prop, so any unupdated callers will fail at compile time. The prompts README has improved documentation on normalization/reference rewriting, though the reference description appears incomplete.

## Findings

### MEDIUM · security

- **Location:** `web/src/features/prompts/components/duplicate-folder.tsx:57–115`
- **Lens:** llm
- **Rationale:** React Hook Form defaultValues are only applied on first render; the initial suggestedTargetPath may be undefined while allNames is loading, leaving targetPath empty or stale. The later useEffect updates only when the form is not dirty, which may still allow an empty target path if the user opens and submits quickly or if suggestedTargetPath changes after the user edits.
- **Suggestion:** Initialize defaultValues with a stable fallback (e.g., `${folderPath}-copy`) and/or call form.reset({targetPath: suggestedTargetPath, ...}) when isOpen changes and data is ready. Also consider disabling submit until suggestedTargetPath is computed and validation passes.

### MEDIUM · authz

- **Location:** `web/src/features/prompts/components/duplicate-prompt.tsx:46–61`
- **Lens:** llm
- **Rationale:** DuplicatePromptForm now requires an additional prop `allPromptNames`. Any callers not updated will break at compile-time/runtime depending on TS enforcement.
- **Suggestion:** Verify all call sites pass allPromptNames (or make the prop optional with an internal query fallback). Add a lightweight component test or TypeScript check in CI to ensure no missing props.

### INFO · general

- **Location:** `web/src/features/prompts/README.md:15–25`
- **Lens:** llm
- **Rationale:** The README now documents normalization and reference rewrite behavior, which is helpful. However, it describes reference rewriting rules that should be guaranteed by the backend, not only by UI preview logic.
- **Suggestion:** Add a short note clarifying that the server enforces the normalization/rewriting rules and link to the server-side implementation/tests once added.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `web/src/features/prompts/components/duplicate-folder.tsx:321–333`
- **Lens:** llm
- **Rationale:** The entitlement check uses `allPromptNames.data.length + duplicateSummary.promptCount >= promptLimit`. If the limit represents the maximum allowed count, the blocking condition is typically `> promptLimit` (allowing exactly-equal) or should be consistent with the rest of the app. As written, it may prevent reaching the limit exactly, or conversely may be off-by-one depending on how the backend enforces it.
- **Merged into:** `llm.duplicate-folder.tsx`

### MEDIUM · security (duplicate)

- **Location:** `web/src/features/prompts/components/duplicate-folder.tsx:129–143`
- **Lens:** llm
- **Rationale:** Client-side normalization (normalizePromptPath) improves UX, but it must not be relied upon for authorization/safety. If the server accepts paths, an attacker can bypass normalization and attempt path confusion (e.g., double slashes, traversal-like segments) via direct API calls.
- **Merged into:** `llm.duplicate-folder.tsx`

### MEDIUM · authz (duplicate)

- **Location:** `web/src/features/prompts/components/duplicate-prompt.tsx:46–61`
- **Lens:** llm
- **Rationale:** DuplicatePromptForm now requires an additional prop `allPromptNames`. Any callers not updated will break at compile-time/runtime depending on TS enforcement.
- **Merged into:** `llm.duplicate-prompt.tsx`
