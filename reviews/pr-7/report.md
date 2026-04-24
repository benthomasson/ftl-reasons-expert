# Code Review Report

**Branch:** benthomasson/ftl-reasons#7
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### reasons_lib/api.py:retract_node
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Logic is correct. `before` snapshot captures all truth values before mutation, then `changed` is iterated to classify transitions. `before.get(nid)` is safe — though `before[nid]` would also work since `changed` only contains existing node IDs (verified in `network.py:76-100`). The `net.nodes[nid]` access on the post-mutation side is also safe for the same reason. Existing tests in `test_api.py` (lines 51-73) only assert on `result["changed"]`, so they still pass but don't verify the new fields.

### reasons_lib/api.py:assert_node
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Identical pattern to `retract_node` — same analysis applies. The `assert_node` path in `network.py:102-120` can trigger `_propagate`, which may flip dependents OUT (e.g., nodes with outlist containing the asserted node). The `went_out` list correctly captures this non-monotonic case, which is the interesting edge case here.

### reasons_lib/cli.py:_print_cascade
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The helper is clean and defensive (uses `.get()` with default `[]`). One concern: the retracted/asserted node itself appears in `went_out`/`went_in` alongside its dependents, so the output mixes the explicitly targeted node with cascade effects. For example, `retract A` prints `Retracted A` then also lists `A` under `Went OUT`. This is mildly redundant but not incorrect. No CLI tests exist to verify the output format.

### reasons_lib/cli.py:cmd_retract
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Output changed from flat `"Retracted: a, b, c"` to structured `"Retracted a"` + cascade breakdown. The new format is more informative — users can distinguish what went OUT vs IN during a retraction cascade. No tests to break.

### reasons_lib/cli.py:cmd_assert
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Mirror of `cmd_retract` changes. Same analysis applies.

### Self-Review
**Limitations:** Could not see `_propagate()` to verify it never returns IDs absent from `net.nodes`. Verified indirectly via `retract` and `assert_node` which only seed `changed` with existing IDs, but `_propagate` could theoretically add synthetic IDs. Also did not have access to the full test suite output to confirm existing tests still pass.

### Feature Requests
- Include the full body of helper methods called by modified functions (e.g., `_propagate`) so reviewers can verify invariants end-to-end
- Show existing test assertions that reference modified return values, to flag tests that should be updated

## gemini [PASS]

### reasons_lib/api.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

The logic correctly records the state of all nodes before an operation and compares it with the state afterward to properly separate the cascade effects into `went_out` and `went_in`. By preserving the `changed` key, this maintains backward compatibility with existing callers and tests. However, no new test assertions were added to `tests/test_api.py` to explicitly verify the correctness of the new `went_out` and `went_in` lists.

### reasons_lib/cli.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The changes cleanly wire up the new API response fields to the CLI output, resulting in a significantly more readable breakdown of cascades (differentiating nodes that went OUT vs those restored IN). It handles empty cascades gracefully and relies safely on `.get()`. There are no automated tests for the CLI layer itself, so these presentation changes remain untested.

### Self-Review
**Limitations:** None. I was able to search the workspace to verify that `truth_value` only uses `"IN"` and `"OUT"` states, confirm that existing API tests don't strictly assert the dictionary structure (so they won't break), and verify that `_propagate` / `retract` methods under the hood behave exactly as expected.

### Feature Requests
- Automatically surface corresponding test files when reviewing API or library changes so that reviewers can verify if test coverage was updated.
