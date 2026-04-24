# Code Review Report

**Branch:** benthomasson/ftl-reasons#31
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### `reasons_lib/network.py:_rebuild_dependents`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly clears all dependents sets then rebuilds from justifications in a two-pass approach. Handles both antecedents and outlist. The `if ant_id in self.nodes` guards correctly skip dangling references. This eliminates three copy-pasted rebuild implementations (storage.py:load, import_agent._fixup_dependents, import_agent._update_node_justifications) and replaces them with a single canonical source. Clean implementation.

### `reasons_lib/network.py:verify_dependents`
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** PARTIAL

The implementation correctly computes expected dependents from justifications and compares against the live index. Error messages are clear and actionable. However, the issue's suggested fix #3 says "Adding an integrity check that validates dependents against justifications **after mutations**" — but `verify_dependents()` is never called from any mutation path or production code. It exists only as a test utility. To fully address the issue, it should either be called after mutations (at least in debug/assert mode) or documented as test-only. As-is, it detects corruption only when someone remembers to call it.

### `reasons_lib/storage.py:load`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Clean replacement of inline rebuild loop with `network._rebuild_dependents()`. The comment is updated to reference "canonical method." Since dependents are not persisted to SQLite (confirmed by examining `save()`), this rebuild-on-load is the correct approach.

### `reasons_lib/import_agent.py:_fixup_dependents`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Simple delegation to `network._rebuild_dependents()`. The docstring is updated to describe the delegation. Callers at lines 312 and 444 (`_import_claims`, `_sync_claims`) are unchanged and continue to work correctly.

### `reasons_lib/import_agent.py:_update_node_justifications`
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctness is improved — the old code did targeted add/discard that could drift from the rebuild logic. The new code is simpler and provably correct. **However, there's a performance regression**: the old code did O(J) targeted updates; the new code does an O(N\*J) full rebuild. In `_sync_claims`, `_update_node_justifications` can be called multiple times per sync (lines 384 and 392 in a loop over claims), meaning the full network gets rebuilt once per changed claim. For large networks this could be significant. Consider either: (a) deferring the rebuild to after the loop and doing one `_fixup_dependents` call, or (b) documenting the trade-off.

### `tests/test_dependents_integrity.py` (new file)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Excellent test coverage organized into logical groups:

- **TestRebuildDependents** (5 tests): empty, premises-only, antecedent, outlist, mixed, stale cleanup
- **TestVerifyDependents** (4 tests): clean detection, extra detection, missing detection, read-only verification
- **TestMutationPathIntegrity** (9 tests): covers add_node, retract/restore, add_justification, supersede, challenge, challenge+defend, convert_to_premise, add_nogood, summarize
- **TestCorruptionAndRepair** (3 tests): detect+repair extra, detect+repair missing, idempotency
- **TestStorageRoundTripDependents** (2 tests): simple and diamond-pattern round-trips
- **TestEdgeCases** (5 tests): dangling refs, duplicate antecedents, self-reference, `_rewrite_dependents` antecedent and outlist

One note on `test_self_referencing_justification` (line 286): it manually constructs a self-referencing node outside the normal API, which is fine for testing the degenerate case but doesn't test whether the API would allow or prevent self-references.

The `test_api_rewrite_dependents_stays_clean` test correctly validates that `_rewrite_dependents` in api.py (which does incremental updates, NOT a full rebuild) leaves the index consistent. Good coverage of a function that wasn't centralized.

### Self-Review
**Limitations:** I was able to read all modified files and their full method implementations, which gave good context. I could not see the full `_sync_claims` loop to count exactly how many times `_update_node_justifications` is called per invocation — the performance concern is based on the pattern described by the explore agent. I also could not run the test suite to verify all 376 lines of test code pass.

### Feature Requests
- Show all callers of modified/deleted functions (e.g., grep for `_fixup_dependents` and `_update_node_justifications` callers) to verify no call sites are broken
- Include the ability to run the test suite and report pass/fail as part of the review
- When a method's docstring makes a claim ("All other rebuild call sites should delegate here"), automatically verify the claim by searching for counter-examples

## gemini [CONCERN]

### reasons_lib/import_agent.py:_update_node_justifications
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

While functionally correct and guaranteed to maintain consistency, replacing the targeted $O(J)$ dependent update logic with a full network $O(V+E)$ rebuild (`network._rebuild_dependents()`) is highly problematic for performance. `_update_node_justifications` is typically called in a loop when synchronizing agent beliefs. This changes the sync complexity to $O(M \times N)$ where M is the number of updated nodes and N is the total network size. This could cause severe performance degradation on large networks. Consider restoring the targeted removal/addition of dependent references, or batching the rebuild so it only occurs once at the end of the import process.

### reasons_lib/import_agent.py:_fixup_dependents
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly refactored to delegate to the new canonical `_rebuild_dependents()` method, reducing duplication. Since this is a one-time fixup per import, the full rebuild is acceptable.

### reasons_lib/network.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly introduces `_rebuild_dependents()` as a single source of truth for rebuilding the index, and `verify_dependents()` to calculate and diff the expected state vs. the live state. The logic flawlessly accounts for both `antecedents` and `outlist` dependencies.

### reasons_lib/storage.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Replaces inline manual rebuilding during load with the new centralized `_rebuild_dependents()` method, improving maintainability.

### tests/test_dependents_integrity.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Very comprehensive and excellent test suite. It effectively addresses the core issue by ensuring all existing network mutation paths (`add_node`, `retract`, `add_justification`, `supersede`, etc.) are exhaustively verified against the `verify_dependents()` integrity check, preventing silent regressions.

### Self-Review
**Limitations:** None. Context was clear, the issue description was well-scoped, and the provided tests verified the behavior thoroughly. The only context missing was whether an `ASSERT` runtime check on mutation paths would have been preferred over just test coverage, but adding comprehensive integrity tests generally satisfies the "adding an integrity check" criteria safely.
