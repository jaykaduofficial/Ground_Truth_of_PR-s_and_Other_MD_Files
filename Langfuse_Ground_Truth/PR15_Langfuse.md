# Golden Comment Verification Report

**PR:** Add prompt version summary metrics (`#1` / `PR15`)
**Repository:** `annoted-pullrequest/langfuse__langfuse-clone__lyxor__PR15__20260603`
**Files changed:** 6
**Verification date:** 2026-07-27

---

## Overview

This report verifies four "golden" review comments against the actual code changes in the pull request diff. Each comment is evaluated strictly against what is visible in the PR — no assumptions are made about code outside the provided diff.

---

## Golden Comment 1

> "The default includes archived prompt versions when the caller omits includeArchived, which can make summary totals include data that normal prompt list views may hide."

| Field | Value |
|---|---|
| **Verdict** | Partially Correct |
| **Confidence** | Medium |

### Reason

The schema default is confirmed: `includeArchived` defaults to `true` in `PromptVersionSummaryInputSchema`, so any caller that omits the field would indeed pull in archived data. That part of the claim is accurate at the schema level.

However, the only caller introduced in this PR (`prompts-table.tsx`) does **not** omit the field — it explicitly passes `includeArchived: false`. So while the underlying risk described is real and would surface for a hypothetical future or alternate caller, the diff itself shows no actual instance of the described bug occurring in current behavior. The comment is more of a defensive/latent-risk flag than a confirmed active issue in this PR.

### Evidence

`packages/shared/src/features/prompts/types.ts`:
```ts
export const PromptVersionSummaryInputSchema = z.object({
  projectId: z.string(),
  name: z.string().optional(),
  label: z.string().optional(),
  includeArchived: z.boolean().default(true),
});
```

`web/src/features/prompts/components/prompts-table.tsx`:
```ts
const promptVersionSummary = api.prompts.versionSummary.useQuery(
  {
    projectId,
    name: searchQuery || undefined,
    includeArchived: false,
  },
  ...
);
```

---

## Golden Comment 2

> "The search term is used as a raw LIKE pattern, so characters such as % and _ act as wildcards and can return a much broader summary than the user intended."

| Field | Value |
|---|---|
| **Verdict** | Correct |
| **Confidence** | High |

### Reason

`input.name` is interpolated directly into an `ILIKE` pattern with no escaping of SQL wildcard characters (`%`, `_`). Since `input.name` originates from `searchQuery` — free-form user-typed text — any `%` or `_` a user types is interpreted by Postgres as a wildcard rather than a literal character, silently broadening the match beyond what the user intended.

### Evidence

`packages/trpc/server/routers/promptRouter.ts`:
```ts
const nameFilter = input.name
  ? Prisma.sql`AND name ILIKE ${`%${input.name}%`}`
  : Prisma.empty;
```

---

## Golden Comment 3

> "This counts every prompt version with any label instead of counting protected labels only, so the protected/labeled metric can be inflated."

| Field | Value |
|---|---|
| **Verdict** | Correct |
| **Confidence** | High |

### Reason

The field is named `protectedLabelCount` and surfaced in the UI as "Labeled versions," but the SQL only checks whether a version has *any* label at all (`cardinality(labels) > 0`) — it does not check for a specific protected label such as production or latest. Any arbitrary custom label a user applies would count toward this metric, so the metric's name and displayed meaning are misleading relative to what it actually measures, effectively inflating the "protected" count.

### Evidence

`packages/trpc/server/routers/promptRouter.ts`:
```ts
COUNT(*) FILTER (WHERE cardinality(labels) > 0) AS "protectedLabelCount"
```

---

## Golden Comment 4

> "The summary query only passes the text search and ignores the current folder path and active table filters, so the displayed summary can disagree with the prompt rows being shown."

| Field | Value |
|---|---|
| **Verdict** | Correct |
| **Confidence** | High |

### Reason

The `versionSummary` query call only forwards `projectId`, `name` (the search text), and a hardcoded `includeArchived: false`. It does not forward `currentFolderPath` or any other active table filters into the query. Notably, `currentFolderPath` **is** used elsewhere in the same component — as the display `label` prop on `PromptVersionSummary` — but is never passed into the actual data query, confirming the summary numbers can diverge from what's shown in a folder-scoped or otherwise filtered table view.

### Evidence

`web/src/features/prompts/components/prompts-table.tsx`:
```ts
const promptVersionSummary = api.prompts.versionSummary.useQuery({
  projectId,
  name: searchQuery || undefined,
  includeArchived: false,
});
...
<PromptVersionSummary
  summary={promptVersionSummary.data}
  isLoading={promptVersionSummary.isLoading}
  label={searchQuery || currentFolderPath || undefined}
/>
```

Contrast — `currentFolderPath` reaches the display label but never reaches the query input, confirming the gap.

---

## Summary

| Metric | Count |
|---|---|
| **Total Correct** | 3 |
| **Total Incorrect / Partially Correct** | 1 (Partially Correct) |

### Overall Quality Assessment

The golden comment set is strong overall — all four comments are grounded in real, verifiable patterns in the diff rather than speculative or off-diff claims.

- **Comments 2, 3, and 4** are clean, high-confidence findings directly traceable to specific lines: an unescaped `ILIKE` wildcard risk, a mislabeled "protected" label metric, and a filter/summary mismatch between the query and the displayed table state.
- **Comment 1** is technically accurate about the schema-level default but overstates the practical impact within this specific PR, since the only current caller sidesteps the default by explicitly passing `includeArchived: false`. A reviewer would flag it as a latent/defensive concern for future callers rather than an active bug in this changeset.

**Recommendation:** Comment 1 would benefit from being scoped as a "defensive default" note rather than an active-bug claim, since no current call site is affected; Comments 2, 3, and 4 are actionable as-is.
