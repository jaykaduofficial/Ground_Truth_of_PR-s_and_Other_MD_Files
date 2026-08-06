# PR Review: lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR37429__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR37429__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR37429__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR37429__20260430@pr:1`
- **Files changed:** 48
- **Route:** code_pr_ensemble
- **Reviewed:** 7/13/2026, 6:16:44 PM

## Metrics

- **Findings:** 4 unique (6 raw) · **Files flagged:** 4 · **Density:** 0.1 findings/file
- **Severity:** critical 0 · high 0 · medium 2 · info 2
- **Files changed:** 48
- **Route:** code_pr_ensemble
- **By category:** general 2 · security 1 · authz 1
- **Top files:** messages_en.properties (1), pom.xml (1), VerifyMessageProperties.java (1), VerifyMessagePropertiesTest.java (1)
- **Sources:** lens 0 · llm 6 · merged 4
- **Duplicates merged:** 2

## Summary

The PR adds an HTML sanitizer and new tests, but the verifier’s English source path is computed via regex replacements on the current file path, which is brittle and could break with path/layout changes. The tests cover several negative cases (illegal tags, no-HTML-allowed, anchor mismatches) but lack a clear positive “valid HTML passes” case, and one message changes placeholder syntax to MessageFormat/choice which may impact formatting/compatibility. It also introduces new compile-scope dependencies (OWASP Java HTML Sanitizer, commons-text), increasing the module’s dependency footprint.

## Findings

### MEDIUM · security

- **Location:** `misc/theme-verifier/src/main/java/org/keycloak/themeverifier/VerifyMessageProperties.java:48–129`
- **Lens:** llm
- **Rationale:** The computed English source file path is derived via regex replacements on the current file path (replacing "resources-community" and forcing "_en.properties"). This is brittle: it can silently point to a non-existent/incorrect file for files without locale suffixes, non-standard naming, or paths not containing the expected segments, causing runtime failures in the verifier rather than a controlled verification message.
- **Suggestion:** Resolve the English bundle via a more robust strategy (e.g., detect locale suffix explicitly, build the base name + "_en.properties" with Path APIs, and if missing, add a verifier message and skip HTML comparison rather than throwing RuntimeException). Add a unit test covering a file without a corresponding English resource and assert graceful handling.

### MEDIUM · authz

- **Location:** `js/apps/account-ui/maven-resources/theme/keycloak.v3/account/messages/messages_en.properties:185–186`
- **Lens:** llm
- **Rationale:** The message format for error-invalid-multivalued-size changes from double-curly placeholders to MessageFormat/choice syntax. If any consumers (UI rendering, formatting utilities) previously expected '{{0}}' style placeholders, this could change runtime formatting behavior and potentially break interpolation or pluralization handling depending on the formatting engine in use.
- **Suggestion:** Verify that both account-ui and admin-ui message rendering uses Java MessageFormat-compatible formatting for these bundles. If not, either revert to the previous placeholder format or update the rendering/formatting code and add an integration/unit test that formats this key with sample values and asserts the expected pluralization output.

### INFO · general

- **Location:** `misc/theme-verifier/src/test/java/org/keycloak/themeverifier/VerifyMessagePropertiesTest.java:29–55`
- **Lens:** llm
- **Rationale:** New tests cover illegal tags, no-HTML-allowed behavior, and anchor mismatches, but there is no positive test proving that allowed HTML (br/p/strong/b) passes, nor that allowed placeholders/choice-format normalization paths do not create false positives.
- **Suggestion:** Add passing tests for: (1) a translation containing only allowed tags when English contains HTML, (2) a file/key using the choice-format pattern (e.g., error-invalid-multivalued-size) to ensure normalizeValue doesn't mask real issues or fail unexpectedly.

### INFO · general

- **Location:** `misc/theme-verifier/pom.xml:72–88`
- **Lens:** llm
- **Rationale:** Adds new compile-scope dependencies (owasp-java-html-sanitizer and commons-text) to the theme-verifier module. This increases the dependency surface and may affect build reproducibility and CVE management.
- **Suggestion:** Confirm versions align with project dependency management/BOM policies and consider using dependencyManagement to control versions. If commons-text is only used for unescapeHtml4, consider whether a lighter dependency (or existing project dependency) can be reused.

## Merged duplicates

### MEDIUM · security (duplicate)

- **Location:** `misc/theme-verifier/src/main/java/org/keycloak/themeverifier/VerifyMessageProperties.java:55–70`
- **Lens:** llm
- **Rationale:** POLICY_SOME_HTML allows only a small subset of elements (br/p/strong/b) but anchor handling is performed separately by removing matched anchor tags from the translation before sanitization, while leaving the English string untouched. This can lead to confusing false positives/negatives and may fail to reliably enforce attribute safety (e.g., href/rel/target) because anchors are not actually allowed by the policy and are handled via string manipulation.
- **Merged into:** `llm.verifymessageproperties.java`

### MEDIUM · general (duplicate)

- **Location:** `misc/theme-verifier/src/main/java/org/keycloak/themeverifier/VerifyMessageProperties.java:89–107`
- **Lens:** llm
- **Rationale:** The diff-reporting logic computes common prefix/suffix and then uses substring(start, len-end). If start becomes equal to length or end surpasses remaining content in certain edge cases (e.g., sanitized shorter than value and all characters match until one string ends), substring indices may become invalid and throw an exception, turning a verification finding into a build failure.
- **Merged into:** `llm.verifymessageproperties.java`
