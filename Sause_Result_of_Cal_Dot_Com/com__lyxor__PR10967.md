# PR Review: lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR10967__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR10967__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR10967__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR10967__20260430@pr:1`
- **Files changed:** 22
- **Route:** code_pr_ensemble
- **Reviewed:** 7/7/2026, 8:54:04 PM

## Metrics

- **Findings:** 3 unique (6 raw) · **Files flagged:** 3 · **Density:** 0.1 findings/file
- **Severity:** critical 0 · high 1 · medium 1 · info 1
- **Files changed:** 22
- **Route:** code_pr_ensemble
- **By category:** authz 2 · general 1
- **Top files:** bookingReminder.ts (1), webhook.e2e.ts (1), CalendarService.ts (1)
- **Sources:** lens 0 · llm 6 · merged 3
- **Duplicates merged:** 3

## Summary

The PR introduces a breaking change by updating `CalendarService.createEvent` to require a new `credentialId` parameter, which may break existing Calendar integrations. It also changes behavior around `destinationCalendar` by normalizing “none selected” from `null` to an empty array and wrapping selected calendars into an array, requiring downstream consumers/tests to align with the new payload shape.

## Findings

### HIGH · authz

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:84–90`
- **Lens:** llm
- **Rationale:** The `createEvent` method signature was changed to require a new `credentialId` parameter, which breaks the Calendar interface/implementations and any callers that still call `createEvent(calEvent)` (non-Google providers in this diff still use the old signature). This likely causes TypeScript interface mismatch and/or runtime call-site errors depending on how `Calendar` is typed and used.
- **Suggestion:** Avoid changing the interface method signature; instead pass `credentialId` via `this.credential` (already on the service) or as an optional parameter with a default. Alternatively, update the shared `Calendar` interface and all provider implementations and call sites consistently.

### MEDIUM · general

- **Location:** `apps/web/pages/api/cron/bookingReminder.ts:104–140`
- **Lens:** llm
- **Rationale:** The reminder email event now always sets `destinationCalendar` to `[]` when none is selected, and wraps the selected calendar into a single-element array. If downstream email/template logic previously expected `null` or a single object, this may change behavior (e.g., conditional checks).
- **Suggestion:** Audit downstream consumers of `evt.destinationCalendar` to ensure they handle an array consistently. Add/adjust tests for reminder email rendering/logic with empty array vs null.

### INFO · authz

- **Location:** `apps/web/playwright/webhook.e2e.ts:246–255`
- **Lens:** llm
- **Rationale:** The webhook payload expectation changed from `destinationCalendar: null` to `destinationCalendar: []`, reflecting an API shape change for consumers of webhook data.
- **Suggestion:** Update webhook schema/types and document the change (or ensure backward compatibility by allowing null or []). Consider a migration period or versioning for webhook consumers.

## Merged duplicates

### HIGH · general (duplicate)

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:236–260`
- **Lens:** llm
- **Rationale:** In `updateEvent`, `selectedCalendar` falls back to `event.destinationCalendar?.find((cal) => cal.externalId === externalCalendarId)?.externalId` when `externalCalendarId` is falsy, but it compares against `externalCalendarId` which is undefined in that branch, resulting in `selectedCalendar` almost always being undefined. Similar logic appears in `deleteEvent`, risking updates/deletes against the wrong calendar (or failing).
- **Merged into:** `llm.calendarservice.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:97–130`
- **Lens:** llm
- **Rationale:** The organizer attendee email is now derived from the first entry of `destinationCalendar` (`mainHostDestinationCalendar`), regardless of which host/credential is creating the event. In multi-host scenarios, the first element may not correspond to the credential used for insertion, leading to incorrect organizer email and potentially confusing attendee/ownership behavior.
- **Merged into:** `llm.calendarservice.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/app-store/googlecalendar/lib/CalendarService.ts:84–170`
- **Lens:** llm
- **Rationale:** Core selection logic now depends on matching `credentialId` to an entry in `destinationCalendar`, but there are no tests shown that validate correct selection when multiple destination calendars exist, or fallback behavior when no matching entry exists (should use primary).
- **Merged into:** `llm.calendarservice.ts`
