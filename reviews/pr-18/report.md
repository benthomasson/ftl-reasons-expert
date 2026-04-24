# Code Review Report

**Branch:** benthomasson/ftl-reasons#18
**Models:** claude
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### reasons_lib/import_agent.py — Refactoring (normalization layer)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The refactoring extracts shared logic into `_normalize_markdown`, `_normalize_json`, `_topo_sort_claims`, `_build_justifications`, and `_import_nogoods`. The normalization correctly preserves all fields from both formats. The `import_agent` and `import_agent_json` public functions now delegate to `_import_claims` using normalized data, maintaining identical behavior. The topological sort preserves the original algorithm. `_build_justifications` correctly places `inactive_id` in outlist (not `active_id` in antecedents), consistent with the documented kill-switch pattern.

### reasons_lib/import_agent.py:_normalize_json
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Line 173 — antecedents are NOT filtered to `node_ids`, unlike the outlist at line 174. This is asymmetric but intentional (antecedents may reference nodes from the local network). However, `_normalize_markdown` at line 128 DOES filter antecedents to `claim_ids`. This inconsistency means JSON import preserves cross-boundary antecedent references while markdown import drops them. Both paths existed in the pre-refactor code so this isn't a regression, but it's a latent inconsistency worth noting.

### reasons_lib/import_agent.py:_sync_claims
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Three concerns:

1. **Ordering of operations** (lines 468-486): `_fixup_dependents` runs before `assert_after` loop, then `recompute_all`, then `retract_after` loop, then `_import_nogoods`. Nogoods are imported AFTER recomputation — this means nogood violations won't be detected during this sync cycle. In contrast, `_import_claims` imports nogoods before recompute. This is inconsistent and could allow contradictory states to persist until the next explicit propagation.

2. **`_retracted` metadata flag** (lines 464-466, 484-485): Nodes removed from the remote that are already OUT get `_retracted = True` to prevent `recompute_all` from resurrecting them. This is sound but creates hidden state that isn't visible in the return dict. If a user later calls `recompute_all` manually, these nodes stay OUT even though their justifications might be valid. This is the correct "remote wins" behavior but should be documented.

3. **Double-counting risk in `retract_after`** (lines 478-486): A node that's both in `remote_ids` (matched as existing, `is_out=True`, added to `retract_after` at line 410) and would otherwise be counted correctly, but `beliefs_retracted` is always incremented at line 486 regardless of whether the node was already OUT. The test `TestSyncNoChanges::test_sync_unchanged` expects `beliefs_retracted == 1` for `gamma-stale` on re-sync, which works because gamma is already OUT and gets `_retracted = True` + counted. This is semantically correct — it reports "how many remote-OUT beliefs were processed" not "how many changed state."

### reasons_lib/import_agent.py:_justifications_match
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Compares justification lists element-wise on type, antecedents, outlist, and label. This correctly detects justification changes for sync updates. Used at lines 407 and 415 to gate `_update_node_justifications`.

### reasons_lib/import_agent.py:_update_node_justifications
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Properly unregisters old dependent references before replacing justifications, then re-registers new ones. This prevents stale dependency edges from causing incorrect propagation.

### reasons_lib/api.py:sync_agent
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly mirrors the structure of `import_agent`. Dispatches to JSON or markdown sync based on file extension. Auto-detects `nogoods.md` adjacent to the beliefs file. Uses `_with_network(write=True)` context manager for ACID guarantees. File reads happen outside the context manager (correct — no need to hold the lock during I/O).

### reasons_lib/cli.py:cmd_sync_agent
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

CLI handler is properly wired in the argument parser and command dispatch table. Output formatting handles all return dict keys. No CLI-level test exists, but the API-level tests cover the core logic. The `--only-in` flag and `--nogoods` option are correctly parsed.

### tests/test_sync_agent.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test suite with 518 lines covering: no-op sync, new belief addition, belief removal, text updates, truth value transitions (IN→OUT, OUT→IN), local retraction override, dependency changes, JSON format sync (add/update/remove/retraction-clear), counting accuracy, first-time sync, idempotency (double-remove, double-sync), and agent revocation cascading after sync. The idempotency tests are particularly valuable — `test_resync_removed_no_double_count` catches a class of bugs that naive implementations would hit. The revocation cascade tests verify that sync doesn't accidentally detach beliefs from the kill-switch mechanism.

### Self-Review
**Limitations:** Could not run the test suite to verify all tests pass. My analysis is based on code reading only. I verified the assertion logic (lines 470-474) is correct by tracing through — nodes are added to `assert_after` at line 424 only when `truth_value == "OUT"`, and assertion happens after `_fixup_dependents`, so the `if truth_value == "OUT"` check at line 473 is properly guarded. However, I cannot confirm there isn't a subtle interaction between `recompute_all` at line 476 and the `retract_after` loop at 478-486 that could cause unexpected state. Running the full test suite would resolve this.

### Feature Requests
- Ability to run tests from within the review to verify claims about correctness
- Show the pre-refactor version of modified functions side-by-side with the new version for refactoring reviews
- Include related test files automatically when implementation files change (test_import_agent.py should have been in observations)
