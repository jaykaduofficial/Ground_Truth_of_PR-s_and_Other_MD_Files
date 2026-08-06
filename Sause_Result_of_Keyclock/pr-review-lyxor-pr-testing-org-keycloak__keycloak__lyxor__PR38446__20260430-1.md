# PR Review: lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR38446__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR38446__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR38446__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR38446__20260430@pr:1`
- **Files changed:** 8
- **Route:** code_pr_ensemble
- **Reviewed:** 7/15/2026, 8:27:44 PM

## Metrics

- **Findings:** 4 unique (6 raw) · **Files flagged:** 4 · **Density:** 0.5 findings/file
- **Severity:** critical 0 · high 1 · medium 1 · info 2
- **Files changed:** 8
- **Route:** code_pr_ensemble
- **By category:** general 2 · security 1 · authz 1
- **Top files:** CredentialHelper.java (1), RecoveryAuthnCodesUtils.java (1), RecoveryAuthnCodesAction.java (1), RecoveryAuthnCodeInputLoginBean.java (1)
- **Sources:** lens 0 · llm 6 · merged 4
- **Duplicates merged:** 2

## Summary

High: `RecoveryAuthnCodesUtils.getCredential(user)` is used but the result is immediately dereferenced via `credentialModelOpt.get()` without checking presence, risking runtime exceptions when no credential exists. Medium: `createRecoveryCodesCredential()` stores `generatedCodes` JSON in a `UserCredentialModel` with an empty identifier, which may cause ambiguity/maintenance issues. Info: recovery code creation now tries federated storage first with fallback to local, and a new public `getCredential(UserModel)` utility was added with call sites updated accordingly.

## Findings

### HIGH · general

- **Location:** `services/src/main/java/org/keycloak/forms/login/freemarker/model/RecoveryAuthnCodeInputLoginBean.java:1–30`
- **Lens:** llm
- **Rationale:** The code now uses RecoveryAuthnCodesUtils.getCredential(user) but immediately calls credentialModelOpt.get() without checking presence. If a user reaches the page without a recovery-codes credential (or if lookup fails), this will throw NoSuchElementException and break the login flow (potentially a 500).
- **Suggestion:** Handle the Optional safely: if empty, set codeNumber to a default, throw a controlled exception mapped to a user-friendly error, or trigger required action/redirect. Add a unit/integration test that exercises the login bean rendering when no recovery-code credential exists.

### MEDIUM · security

- **Location:** `server-spi-private/src/main/java/org/keycloak/utils/CredentialHelper.java:109–135`
- **Lens:** llm
- **Rationale:** createRecoveryCodesCredential() serializes generatedCodes to JSON and stores it in a UserCredentialModel with an empty id and the type set, then calls user.credentialManager().updateCredential(). This may not be compatible with all user storage implementations (some may not support updateCredential for unknown types, may require a specific credential format, or may treat empty id specially), and can lead to inconsistent behavior or duplicate credentials if the provider interprets update as create.
- **Suggestion:** Define/validate the expected contract for federated providers (e.g., use createCredential or a dedicated SPI method if available). Consider checking for existing recovery-code credential and replacing it explicitly. Add integration tests with a federated storage provider verifying that creation and subsequent validation/consumption of recovery codes works and does not create duplicates.

### INFO · general

- **Location:** `services/src/main/java/org/keycloak/authentication/requiredactions/RecoveryAuthnCodesAction.java:80–140`
- **Lens:** llm
- **Rationale:** Behavior changed from always creating the credential in local storage to attempting federated storage first and falling back. Without dedicated tests, regressions are likely (e.g., federated path not persisting correctly, fallback not triggered, or codes not consumable).
- **Suggestion:** Add integration tests covering: (1) creation into federated storage when updateCredential returns true, (2) fallback to local when updateCredential returns false, and (3) authentication using a recovery code after each storage path.

### INFO · authz

- **Location:** `server-spi/src/main/java/org/keycloak/models/utils/RecoveryAuthnCodesUtils.java:1–70`
- **Lens:** llm
- **Rationale:** A new public utility method getCredential(UserModel) is introduced and existing call sites are updated to use it. While not a breaking change by itself, it alters credential lookup semantics (federated-first) which can change behavior for deployments with both federated and local credentials present.
- **Suggestion:** Document the precedence rule (federated over local) and consider how conflicts should be resolved if both exist. Add a test to confirm deterministic selection when both federated and local recovery-code credentials exist.

## Merged duplicates

### MEDIUM · security (duplicate)

- **Location:** `server-spi-private/src/main/java/org/keycloak/utils/CredentialHelper.java:109–135`
- **Lens:** llm
- **Rationale:** Recovery codes are serialized as raw JSON and passed into updateCredential for user storage. Depending on the storage provider, this may store recovery codes in plaintext outside of Keycloak's credential hashing/encryption protections, increasing the impact of a storage breach.
- **Merged into:** `llm.credentialhelper.java`

### MEDIUM · general (duplicate)

- **Location:** `server-spi-private/src/main/java/org/keycloak/utils/CredentialHelper.java:109–135`
- **Lens:** llm
- **Rationale:** The method obtains a CredentialProvider via session.getProvider(CredentialProvider.class, "keycloak-recovery-authn-codes") but does not null-check it. If the provider id changes, is disabled, or not registered in a given distribution, this will cause a NullPointerException when falling back to local creation.
- **Merged into:** `llm.credentialhelper.java`
