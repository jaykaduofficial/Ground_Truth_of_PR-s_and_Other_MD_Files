# Golden Comment Validation Report

**PR:** Feature flag: rolling-updates
**Repo:** keycloak/keycloak
**PR ID:** #36882
**Source PDF:** `keycloak__keycloak__lyxor__PR36882__20260430`
**Files changed:** 11
**Commits:** 5
**Validation date:** 2026-07-01

---

## Golden Comment 1

**Comment:** Incorrect method call for exit codes. The `picocli.exit()` method calls `System.exit()` directly, which is problematic.

- **Verdict:** ✅ Correct
- **Reason:** Both `UpdateCompatibilityCheck.run()` and `UpdateCompatibilityMetadata.run()` add an early-exit branch that calls `picocli.exit(CompatibilityResult.FEATURE_DISABLED)` directly from inside the command's business logic, before `printPreviewWarning()`/`validateConfig()` run. Picocli's `CommandLine.exit(int)` is a thin wrapper that terminates the JVM via `System.exit(status)` directly — intended as a convenience for a top-level `main()` after `execute()` returns, not for use inside a subcommand's `run()` method. Calling it mid-command:
  - Kills the JVM abruptly instead of letting the exit code propagate back through picocli's normal `execute()` return flow.
  - Bypasses cleanup/shutdown hooks and any subsequent command processing.
  - Makes the method difficult to unit test in isolation (tests in this PR route around this via the `@Launch(...)` distribution-test harness rather than a plain unit test).

  The identical pattern appears in two separate files, reinforcing that this is a real, reproducible design issue rather than an edge case.

- **Evidence:**
  - File: `runtime/cli/command/UpdateCompatibilityCheck.java`
    ```java
    @Override
    public void run() {
        if (!Profile.isFeatureEnabled(Profile.Feature.ROLLING_UPDATES)) {
            printFeatureDisabled();
            picocli.exit(CompatibilityResult.FEATURE_DISABLED);
            return;
        }
        printPreviewWarning();
        validateConfig();
        ...
    ```
  - File: `runtime/cli/command/UpdateCompatibilityMetadata.java` (identical pattern)
    ```java
    @Override
    public void run() {
        if (!Profile.isFeatureEnabled(Profile.Feature.ROLLING_UPDATES)) {
            printFeatureDisabled();
            picocli.exit(CompatibilityResult.FEATURE_DISABLED);
            return;
        }
        printPreviewWarning();
        validateConfig();
    ```

- **Confidence:** Medium — the diff does not show the declaration/type of the `picocli` field itself (outside the changed hunks), so it can't be 100% confirmed to be a `picocli.CommandLine` instance vs. a custom wrapper. However, the field name and described `System.exit()`-calling behavior strongly match picocli's documented `CommandLine.exit(int)` API, and the call-then-`return` usage pattern is consistent with the comment's concern.

---

## Summary

| Metric | Count |
|---|---|
| Total golden comments evaluated | 1 |
| Correct | 1 |
| Incorrect | 0 |
| Partially Correct | 0 |

**Overall quality assessment:**
The single golden comment is accurate and identifies a genuine, duplicated design issue (mid-method `System.exit()` via picocli's `exit()` helper) introduced consistently across two new feature-gating checks in this PR. The confidence is Medium rather than High only because the exact type of the `picocli` field isn't visible in the diff itself — it must be inferred from naming and behavior.

**Note (informational, not part of scoring):**
This PR is otherwise mostly documentation/config plumbing for the new `rolling-updates` feature flag (Profile.java enum addition, AsciiDoc guide updates, Dockerfile build arg, exit-code renumbering in `CompatibilityResult`). No other functional defects were identified in the visible diff beyond the one golden comment already covers, so a single golden comment for this PR is reasonably proportionate to its actual code-review surface area.
