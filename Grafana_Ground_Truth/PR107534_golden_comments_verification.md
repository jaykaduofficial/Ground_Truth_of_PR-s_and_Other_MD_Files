# Golden Comments Evaluation Report

**PR:** Query splitting: Interpolate queries at the start of the process (#1)
**Repo:** lyxor-pr-testing-org/grafana__grafana__lyxor__PR107534__20260504

---

## Golden Comment 1

**Comment:** The applyTemplateVariables method is called with request.filters as the third parameter, but this parameter is not used in the corresponding test setup.

**Verdict:** Correct

**Reason:** In both `querySplitting.ts` and `shardQuerySplitting.ts`, the new query-interpolation step calls `datasource.applyTemplateVariables(query, request.scopedVars, request.filters)`, passing `request.filters` as the third argument. However, in the test files updated alongside this change, the mock implementations of `applyTemplateVariables` only account for the first (and in one case, conceptually the second) parameter, ignoring `filters` entirely. In `shardQuerySplitting.test.ts`, the mock is defined as `jest.fn().mockImplementation((query: LokiQuery) => { query.expr = query.expr.replace('$SELECTOR', '{a="b"}'); return query; })` — a single-parameter function that never reads or asserts on a `filters` argument. Similarly, in `querySplitting.test.ts`, `createLokiDatasource({ replace, getVariables })` is configured without any mock or assertion tied to `filters`, so nothing in the test setup verifies that `request.filters` is correctly threaded through to `applyTemplateVariables`. This means the third parameter is passed in production code but has no corresponding test coverage confirming it's used or passed correctly.

**Evidence:**
```ts
// querySplitting.ts / shardQuerySplitting.ts
.map((query) => datasource.applyTemplateVariables(query, request.scopedVars, request.filters));
```
```ts
// shardQuerySplitting.test.ts
datasource.applyTemplateVariables = jest.fn().mockImplementation((query: LokiQuery) => {
  query.expr = query.expr.replace('$SELECTOR', '{a="b"}');
  return query;
});
```
The mock signature accepts only `query`, with no handling or assertions involving `request.filters`.

**Confidence:** High — the mismatch between the three-argument call site and the single-argument mock implementation is directly visible in the diff.

---

## Summary

| Metric | Count |
|---|---|
| Total correct golden comments | 1 |
| Total incorrect / partially correct | 0 |

**Overall quality assessment:** This is a valid, accurate observation about test coverage rather than a functional bug — it correctly identifies that the new `request.filters` argument passed to `applyTemplateVariables` isn't validated or even consumed by the corresponding test mocks in either `querySplitting.test.ts` or `shardQuerySplitting.test.ts`. It's a legitimate, narrowly-scoped code review comment (more of a "missing test assertion" flag than a defect report), and it's well-supported by the diff.
