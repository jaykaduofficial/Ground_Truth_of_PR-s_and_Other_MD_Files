# PR Review: lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR7232__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR7232__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR7232__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR7232__20260430@pr:1`
- **Files changed:** 10
- **Route:** code_pr_ensemble
- **Reviewed:** 7/7/2026, 8:51:02 PM

## Metrics

- **Findings:** 4 unique (8 raw) · **Files flagged:** 4 · **Density:** 0.4 findings/file
- **Severity:** critical 0 · high 2 · medium 1 · info 1
- **Files changed:** 10
- **Route:** code_pr_ensemble
- **By category:** general 3 · security 1
- **Top files:** handleCancelBooking.ts (1), handleNewBooking.ts (1), scheduleEmailReminders.ts (1), WorkflowStepContainer.tsx (1)
- **Sources:** lens 0 · llm 8 · merged 4
- **Duplicates merged:** 4

## Summary

Key concern: cancellation handling no longer deletes `workflowReminder` DB rows (`handleCancelBooking.ts`), instead relying on an external delete, while `scheduleEmailReminders.ts` introduces a high‑risk cleanup that deletes reminders marked `cancelled: true` scheduled within the next hour (could cause unintended deletions or race conditions). Reschedule logic now fetches `workflowReminders: true` (`handleNewBooking.ts`), which may meaningfully increase query/payload cost. UI changes in `WorkflowStepContainer.tsx` adjust sender verification conditionals; verify `isSenderIdNeeded` behavior is still correct.

## Findings

### HIGH · general

- **Location:** `packages/features/bookings/lib/handleCancelBooking.ts:483–506`
- **Lens:** llm
- **Rationale:** Previously, workflowReminder rows were deleted from the database on cancellation; now the code only calls external deletion functions and does not remove or mark reminders in DB here. If the external deletion fails or is a no-op (missing referenceId), reminders can remain persisted and may still be picked up by other schedulers/processes, leading to duplicate sends or repeated cancellation attempts.
- **Suggestion:** Restore DB-side state changes in this handler (either delete the workflowReminder rows or set `cancelled: true` + `scheduled: false`/nullify `referenceId`) and make the external cancellation best-effort. Ensure the DB update is part of the same Promise.all worklist so failures can be surfaced/retried deterministically.

### HIGH · security

- **Location:** `packages/features/ee/workflows/api/scheduleEmailReminders.ts:39–86`
- **Lens:** llm
- **Rationale:** This endpoint now deletes workflowReminder rows that are `cancelled: true` and scheduled within the next hour. That is an API/behavior change: it mutates DB state and cancels provider sends on every run. If `cancelled` is set incorrectly or used for other semantics, reminders could be deleted prematurely. Additionally, if `referenceId` is null/empty, the provider cancel call may fail and the reminder will still be deleted, potentially losing traceability/audit.
- **Suggestion:** Gate deletion on presence/validity of `referenceId` and/or a `scheduled` flag; otherwise only mark as cancelled and keep the record. Consider using `updateMany` to mark as cancelled and delete only after confirmed provider cancellation, or store cancellation result/status. Add a feature flag or tighten query to `cancelled: true, scheduled: true, referenceId: { not: null }`.

### MEDIUM · general

- **Location:** `packages/features/bookings/lib/handleNewBooking.ts:759–776`
- **Lens:** llm
- **Rationale:** Adding `workflowReminders: true` to the rescheduled booking fetch may significantly increase payload size and query cost, and changes behavior for this endpoint. Also, the subsequent `forEach` cancellation inside a try/catch does not await potential async calls, so failures may not be caught and cancellations may not complete before reschedule proceeds.
- **Suggestion:** Fetch only needed fields (`select: { workflowReminders: { select: { id: true, method: true, referenceId: true }}}`) and convert the loop to `for...of` with `await`, or collect promises and `await Promise.all` so the try/catch is effective.

### INFO · general

- **Location:** `packages/features/ee/workflows/components/WorkflowStepContainer.tsx:387–470`
- **Lens:** llm
- **Rationale:** UI logic change removes `isSenderIdNeeded` from the conditional and reorganizes verification UI. If sender ID is still required for some workflows, this may regress configuration UX and allow saving invalid steps.
- **Suggestion:** Confirm whether `isSenderIdNeeded` is intentionally deprecated; if not, reintroduce the combined conditional or add separate UI handling for sender ID requirements. Add a UI test to ensure required fields appear for each workflow method.

## Merged duplicates

### MEDIUM · general (duplicate)

- **Location:** `packages/features/bookings/lib/handleCancelBooking.ts:486–501`
- **Lens:** llm
- **Rationale:** The new cancellation calls are made without awaiting. If deleteScheduledEmailReminder/deleteScheduledSMSReminder are async (likely, since they call providers), errors may be unhandled and the request may complete before cancellations are actually performed.
- **Merged into:** `llm.handlecancelbooking.ts`

### MEDIUM · security (duplicate)

- **Location:** `packages/features/ee/workflows/api/scheduleEmailReminders.ts:74–86`
- **Lens:** llm
- **Rationale:** Errors are logged with `console.log` and may include provider/client error details; depending on runtime logging, this can leak operational data. Also, the catch wraps both provider cancellations and DB deletions; partial failures could leave inconsistent state without structured logging/alerting.
- **Merged into:** `llm.scheduleemailreminders.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/features/ee/workflows/api/scheduleEmailReminders.ts:52–63`
- **Lens:** llm
- **Rationale:** The query cancels reminders scheduled within the next hour but does not filter by `method` (email) despite being in scheduleEmailReminders. If SMS reminders share the same table and can be `cancelled: true` with scheduledDate set, this endpoint may attempt to cancel non-email reminders via the email provider API.
- **Merged into:** `llm.scheduleemailreminders.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/features/bookings/lib/handleCancelBooking.ts:483–506`
- **Lens:** llm
- **Rationale:** Cancellation/reschedule reminder behavior is regression-prone, and this PR changes core deletion semantics. Without tests, it's easy to reintroduce the bug (reminders still sent) or introduce new ones (reminders deleted incorrectly).
- **Merged into:** `llm.handlecancelbooking.ts`
