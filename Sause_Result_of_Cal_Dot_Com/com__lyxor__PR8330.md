# PR Review: lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR8330__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR8330__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR8330__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR8330__20260430@pr:1`
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **Reviewed:** 7/7/2026, 8:47:17 PM

## Metrics

- **Findings:** 3 unique (6 raw) · **Files flagged:** 4 · **Density:** 1 findings/file
- **Severity:** critical 0 · high 1 · medium 3
- **Files changed:** 4
- **Route:** code_pr_ensemble
- **By category:** general 3 · authz 1
- **Top files:** getSchedule.test.ts (1), slots.ts (1), slots.ts (1), schedule.d.ts (1)
- **Sources:** lens 0 · llm 6 · merged 4
- **Duplicates merged:** 2

## Summary

The PR has a high-severity bug where date override logic compares two `dayjs().add()` objects with `===`, which checks object identity and will never behave as intended. It also has multiple medium-severity concerns: timezone offset calculation relies on parsing `Date.toString()` (fragile/locale-dependent), test coverage is too narrow (no DST and offset edge cases), and adding an optional `timeZone` to `TimeRange` changes schedule object shapes and may break downstream consumers despite being optional.

## Findings

### HIGH · general

- **Location:** `packages/trpc/server/routers/viewer/slots.ts:104–169`
- **Lens:** llm
- **Rationale:** The date override comparison uses `dayjs(...).add(...) === dayjs(...).add(...)`, which compares object identity and will always be false. This makes the "start equals end" (likely meaning all-day/unavailable or a special case) branch ineffective and can incorrectly mark slots available/unavailable.
- **Suggestion:** Replace the equality check with a value comparison, e.g. `dayjs(...).add(...).isSame(dayjs(...).add(...))` or compare `.valueOf()`/`.toISOString()`.

### MEDIUM · general

- **Location:** `packages/lib/slots.ts:214–233`
- **Lens:** llm
- **Rationale:** Timezone offsets are calculated via `dayjs(override.start.toString()).tz(...)`, which depends on `Date.toString()` parsing and local environment behavior; this can introduce subtle offset/DST bugs. Also, `utc().add(offset)` is recomputed multiple times for the same value, increasing complexity and risk of inconsistency.
- **Suggestion:** Avoid `toString()` and use `dayjs(override.start).tz(...)` (or `dayjs.tz(override.start, ...)`) consistently; compute shifted start/end once (e.g. `const shiftedStart = dayjs(override.start).utc().add(offset,'minute')`) and reuse for hour/minute extraction.

### MEDIUM · general

- **Location:** `apps/web/test/lib/getSchedule.test.ts:784–807`
- **Lens:** llm
- **Rationale:** The added test covers a single different-timezone scenario but does not cover DST transitions, organizer/invitee offset differences that cross midnight, or multiple overrides. Given the new offset math and date-boundary logic, these are high-risk edge cases.
- **Suggestion:** Add tests for: (1) override that crosses midnight in one timezone but not the other, (2) DST change dates for organizer/invitee timezones, (3) override start==end behavior, and (4) slot end extending past working hours (to validate the corrected end-time logic).

### MEDIUM · authz

- **Location:** `packages/types/schedule.d.ts:1–9`
- **Lens:** llm
- **Rationale:** Adding `timeZone?: string` to `TimeRange` changes the shape of schedule-related objects. While optional, downstream code that assumes an exact type or serializes/deserializes strict schemas may break or ignore the new field, and server/client expectations can diverge.
- **Suggestion:** Ensure any zod/schema validation and API serialization layers tolerate/propagate `timeZone`, and add a changelog note (or version bump) if this type is part of a public contract.

## Merged duplicates

### HIGH · general (duplicate)

- **Location:** `packages/trpc/server/routers/viewer/slots.ts:145–166`
- **Lens:** llm
- **Rationale:** Working-hours validation computes `const start = ...; const end = ...;` but `end` is incorrectly set equal to `start` (same expression). This fails to validate the slot end time and can allow slots that extend beyond working hours (or incorrectly reject valid ones).
- **Merged into:** `llm.slots.ts`

### MEDIUM · general (duplicate)

- **Location:** `packages/trpc/server/routers/viewer/slots.ts:107–123`
- **Lens:** llm
- **Rationale:** UTC offset handling uses `utcOffset() * -1` and then adds minutes to the date (`add(utcOffset, 'minutes')`). The sign convention is easy to get wrong (and likely differs from dayjs' `utcOffset` semantics), which can shift override dates to the wrong day, especially around DST boundaries.
- **Merged into:** `llm.slots.ts`
