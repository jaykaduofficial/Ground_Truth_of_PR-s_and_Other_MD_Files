# PR Review: lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR33832__20260430 #1

- **PR:** https://github.com/lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR33832__20260430/pull/1
- **Base scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR33832__20260430@main`
- **PR scope:** `code:lyxor-pr-testing-org/keycloak__keycloak__lyxor__PR33832__20260430@pr:1`
- **Files changed:** 12
- **Route:** code_pr_ensemble
- **Reviewed:** 7/14/2026, 2:09:16 PM

## Metrics

- **Findings:** 4 unique (5 raw) · **Files flagged:** 4 · **Density:** 0.3 findings/file
- **Severity:** critical 0 · high 1 · medium 3
- **Files changed:** 12
- **Route:** code_pr_ensemble
- **By category:** general 2 · authz 1 · security 1
- **Top files:** pom.xml (1), AuthzClient.java (1), ASN1Decoder.java (1), ASN1Encoder.java (1)
- **Sources:** lens 0 · llm 5 · merged 4
- **Duplicates merged:** 1

## Summary

The PR introduces several crypto-related risks: `AuthzClient.create()` now calls `CryptoIntegration.init(...)`, adding a new global/static initialization with potential side effects. More critically, `ASN1Decoder.readSequence()` can loop forever on indefinite-length encodings (when `readLength()` returns `-1`), and `ASN1Encoder.writeDerSeq` uses a DER sequence tag without enforcing DER constraints (e.g., INTEGER encoding/sequence validity). It also adds JUnit/Hamcrest test deps without accompanying tests despite the high-risk crypto/provider changes.

## Findings

### HIGH · security

- **Location:** `authz/client/src/main/java/org/keycloak/authorization/client/util/crypto/ASN1Decoder.java:38–92`
- **Lens:** llm
- **Rationale:** readSequence() does not handle indefinite-length encoding (readLength() returns -1) and will loop forever because length stays negative and the condition is while (length > 0) (it will return empty list without consuming content) or will not properly parse, depending on callers. Additionally, it subtracts bytes.length (which includes tag+len+content) from 'length' (which is content length), causing incorrect accounting and potential premature termination or over-read if used with nested structures.
- **Suggestion:** Reject indefinite-length encoding explicitly for DER (throw IOException when length == -1) and in readSequence() subtract only the content length (or track consumed bytes properly). Add unit tests for sequences with multiple elements, nested sequences, long-form lengths, and invalid/indefinite lengths.

### MEDIUM · authz

- **Location:** `authz/client/src/main/java/org/keycloak/authorization/client/AuthzClient.java:91–95`
- **Lens:** llm
- **Rationale:** Calling CryptoIntegration.init(...) during AuthzClient.create(Configuration) introduces a new global/static initialization side effect that can change behavior for existing consumers (provider selection, classloading, security provider registration) and can be problematic in environments with strict classloaders (OSGi, app servers) or where initialization order matters.
- **Suggestion:** Consider lazy/guarded initialization (idempotent check) or moving init to a dedicated opt-in path; add documentation/release note about the new side effect. Add tests verifying multiple create() calls and different classloaders do not break or reinitialize unexpectedly.

### MEDIUM · general

- **Location:** `authz/client/src/main/java/org/keycloak/authorization/client/util/crypto/ASN1Encoder.java:46–86`
- **Lens:** llm
- **Rationale:** writeDerSeq uses CONSTRUCTED|SEQUENCE tag but does not enforce DER constraints for INTEGER encoding and sequence content. If this is used for key material, any divergence from strict DER may cause interop issues or subtle parsing bugs in other libraries.
- **Suggestion:** Clarify intended encoding (DER vs BER) and enforce DER rules (e.g., minimal length encoding, canonical forms). Add round-trip tests (encode->decode) and interop tests against JCA/BouncyCastle for the specific structures being encoded.

### MEDIUM · general

- **Location:** `authz/client/pom.xml:58–74`
- **Lens:** llm
- **Rationale:** JUnit and Hamcrest are added as test dependencies, but no tests are shown in the diff. Crypto/provider changes are high-risk and should be covered to prevent regressions and security-sensitive parsing issues.
- **Suggestion:** Add unit tests for ASN1Encoder/ASN1Decoder (valid/invalid cases) and for AuthzClientCryptoProvider behavior, plus a test ensuring AuthzClient.create triggers CryptoIntegration init without failures. Ensure tests run under the module's build (surefire) and are compatible with the project’s chosen JUnit version.

## Merged duplicates

### MEDIUM · security (duplicate)

- **Location:** `authz/client/src/main/java/org/keycloak/authorization/client/util/crypto/ASN1Decoder.java:120–186`
- **Lens:** llm
- **Rationale:** The decoder operates on attacker-controlled input in typical crypto/key parsing scenarios; while there are some bounds checks, length handling is fragile (e.g., length >= limit uses the original byte-array size rather than remaining bytes; mark()/reset() manipulates count which is reused for length computations). These issues can lead to parsing inconsistencies and potential DoS via crafted inputs (exceptions, excessive loops, or heavy allocations).
- **Merged into:** `llm.asn1decoder.java`
