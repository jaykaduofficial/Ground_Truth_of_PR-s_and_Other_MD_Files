# Golden Comment Verification Report

**PR:** Async import of the appStore packages (`#1` / `PR8087`)
**Repository:** `lyxor-pr-testing-org/cal_dot_com__cal.com__lyxor__PR8087__20260430`
**Files changed:** 12
**Verification date:** 2026-06-26

---

## Overview

This report verifies two "golden" review comments against the actual code changes in the pull request diff. Each comment is evaluated strictly against what is visible in the PR — no assumptions are made about code outside the provided diff.

---

## Golden Comment 1

> "Consider adding try-catch around the await to handle import failures gracefully"

| Field | Value |
|---|---|
| **Verdict** | Partially Correct |
| **Confidence** | Medium |

### Reason

The PR converts `appStore` (in `packages/app-store/index.ts`) from synchronous re-exports into a map of dynamic `import()` calls, and every consumer that previously did `appStore[key]` now does `await appStore[key]`. Dynamic `import()` can reject (e.g. module fails to load or parse), so this is a real new failure surface.

However, the comment doesn't name a specific file or line, and coverage is inconsistent across call sites:

- Some `await appStore[...]` / `await item.calendar` usages **are** inside an existing `try { ... }` block (e.g. `getConnectedCalendars` in `CalendarManager.ts`).
- Others appear to have **no** local try-catch around them (e.g. `getCalendar.ts`, `handlePayment.ts`, `deletePayment.ts`).

Because the comment is generic rather than anchored to one location, it can't be confirmed as fully accurate or fully inaccurate — it correctly flags a class of risk introduced by the PR, but doesn't pinpoint where the gap actually is.

### Evidence

- `packages/app-store/index.ts`: `applecalendar: import("./applecalendar")`, etc. — appStore values are now Promises.
- `packages/app-store/_utils/getCalendar.ts`: `const calendarApp = await appStore[...]` — no local try-catch visible.
- `packages/lib/payment/handlePayment.ts`: `const paymentApp = await appStore[...]` — no try-catch visible around it.
- `packages/core/CalendarManager.ts`: `const calendar = await item.calendar;` — **is** inside `try { ... }` (covered).

---

## Golden Comment 2

> "The code uses forEach with async callbacks, which causes asynchronous operations (e.g., calendar/video event deletions, payment refunds) to run concurrently without being awaited. This 'fire-and-forget' behavior leads to unhandled promise rejections, race conditions, and incomplete cleanup, as surrounding try-catch blocks cannot properly handle errors from these unawaited promises. Replace forEach with for...of loops or Promise.all() with map() to ensure proper sequential execution and error handling."

| Field | Value |
|---|---|
| **Verdict** | Correct |
| **Confidence** | High |

### Reason

This is a genuine bug introduced by the PR. Because `getCalendar` was changed to `async`, several `forEach` callbacks that call it had to become `async` themselves — but `Array.prototype.forEach` never awaits its callback's returned promise. The result: the outer function (and its surrounding `try { }`) continues without waiting for the callback to finish, and any rejection inside the async callback becomes an **unhandled promise rejection** instead of being caught.

This exact pattern shows up in multiple places in the diff:

1. `packages/app-store/vital/lib/reschedule.ts` — `forEach(async (bookingRef) => {...})` wraps `await getCalendar(...)` and a calendar/video deletion, inside a `try {}` that can't catch it.
2. `packages/app-store/wipemycalother/lib/reschedule.ts` — identical pattern.
3. `packages/trpc/server/routers/viewer/bookings.tsx` — same pattern for booking reschedule cleanup.
4. `packages/features/bookings/lib/handleNewBooking.ts` — `.filter(...).forEach(async (credential) => { const calendar = await getCalendar(credential); ... })` for recurring-event calendar cleanup.

Notably, the PR author **did** apply the correct fix elsewhere — in `handleCancelBooking.ts`, a `.forEach(...)` was deliberately replaced with a `for (const credential of calendarCredentials) { ... }` loop. This confirms the safer pattern was known and used inconsistently, strengthening confidence that the flagged sites are a real oversight rather than an intentional design choice.

### Evidence

`packages/app-store/vital/lib/reschedule.ts`:
```ts
try {
  bookingRefsFiltered.forEach(async (bookingRef) => {
    if (bookingRef.uid) {
      if (bookingRef.type.endsWith("_calendar")) {
        const calendar = await getCalendar(credentialsMap.get(bookingRef.type));
        return calendar?.deleteEvent(bookingRef.uid, builder.calendarEvent);
      } else if (bookingRef.type.endsWith("_video")) {
        return deleteMeeting(credentialsMap.get(bookingRef.type), bookingRef.uid);
      }
    }
  });
```

`packages/trpc/server/routers/viewer/bookings.tsx`:
```ts
bookingRefsFiltered.forEach(async (bookingRef) => {
  ...
  const calendar = await getCalendar(credentialsMap.get(bookingRef.type));
  return calendar?.deleteEvent(...);
});
```

`packages/features/bookings/lib/handleNewBooking.ts`:
```ts
.filter((credential) => credential.type.endsWith("_calendar"))
.forEach(async (credential) => {
  const calendar = await getCalendar(credential);
  ...
});
```

Contrast — correct fix applied elsewhere in `packages/features/bookings/lib/handleCancelBooking.ts`:
```ts
const calendarCredentials = bookingToDelete.user.credentials.filter((credential) =>
  credential.type.endsWith("_calendar")
);
for (const credential of calendarCredentials) {
  const calendar = await getCalendar(credential);
  apiDeletes.push(calendar?.deleteEvent(uid, evt, externalCalendarId) as Promise<unknown>);
}
```

---

## Summary

| Metric | Count |
|---|---|
| **Total Correct** | 1 |
| **Total Incorrect / Partially Correct** | 1 (Partially Correct) |

### Overall Quality Assessment

The golden comment set is of mixed but reasonable quality:

- **Comment 2** is a strong, well-supported finding. It identifies a genuine bug class introduced specifically by this PR (the sync→async conversion of `getCalendar`/`getVideoAdapters` forced several `forEach` callbacks to become `async`, reintroducing a fire-and-forget hazard), explains the mechanism correctly, and proposes the exact fix the PR itself uses inconsistently in other files.
- **Comment 1** points at a real but non-specific risk (dynamic `import()` can fail and isn't uniformly wrapped in try-catch). Without a file/line anchor, it can't be confirmed against one true gap — some of the plausible target sites already have error handling, others don't — so it lands as Partially Correct rather than fully verified.

**Recommendation:** Comment 1 would benefit from being anchored to a specific call site (e.g. `getCalendar.ts` or `handlePayment.ts`) to be fully actionable and verifiable; Comment 2 is actionable as-is.
