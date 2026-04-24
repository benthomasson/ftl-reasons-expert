# Code Review Report

**Branch:** benthomasson/ftl-reasons#12
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## Disagreements

The following changes have different verdicts across models:

- **reasons_lib/network.py:recompute_all** [LOW]: claude: CONCERN, gemini: PASS

## claude [CONCERN]

### reasons_lib/network.py:recompute_all
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The fixpoint loop is sound for the monotone case, and the iteration cap (`len(self.nodes) + 1`) prevents infinite oscillation in non-monotone graphs (outlist cycles). However, `recompute_all` does not distinguish between nodes that are OUT because their justifications are invalid vs. nodes that were **explicitly retracted** via `retract()`. It calls `_compute_truth`, which only checks justifications — it has no concept of explicit retraction. This means `recompute_all()` will **undo intentional retractions** on any derived node whose justification happens to still be valid. This is the root of the gamma-stale semantic issue (see import_agent review). As a general-purpose `Network` method, this is dangerous: any caller using `retract()` to override a derived node's truth value will have that override silently reversed by a subsequent `recompute_all()`.

### reasons_lib/import_agent.py (line 168)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

`recompute_all()` is called after all nodes are imported and STALE/OUT nodes are retracted (lines 143-146). This resurrects gamma-stale: it was parsed as STALE, explicitly retracted on line 145, then `recompute_all()` on line 168 re-evaluates its justification (antecedent `active` is IN), finds it valid, and flips it back to IN. The staleness reason ("gamma is actually third") is metadata — it doesn't affect the justification graph. This means the import flow now **silently overrides the source agent's STALE/OUT designations** for any node whose import-generated justification is satisfied. The `claims_retracted` count in the result (still 1 for gamma-stale) is now misleading since the retraction is immediately undone. If the intent is to honor the justification graph over source status, the retraction on line 145 is wasted work and should be removed. If the intent is to honor source status, `recompute_all()` needs to skip explicitly-retracted nodes.

### reasons_lib/cli.py:cmd_propagate
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Clean extraction — replaces inline single-pass logic with `net.recompute_all()`, which is strictly better (fixpoint iteration catches cascading changes the old single-pass missed). The old inline code had the same semantic issue (no retraction awareness), so this isn't a regression.

### reasons_lib/cli.py:cmd_import_agent (lines 390-391)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Uses `.get('claims_propagated')` defensively, correct formatting consistent with surrounding output. Minor note: the label says "truth values recomputed" which is accurate.

### tests/test_import_agent.py (modified tests)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The updated assertions in `test_import_agent_retract_premise_cascades` (2→3) and `test_import_agent_retract_premise_actually_cascades` (3→4) are internally consistent with the new behavior: gamma-stale is now IN after import, so retracting `active` cascades to it too. The comments explain this. However, the tests encode and normalize what I consider a semantic concern — a STALE belief being silently resurrected. If this is the intended design, the tests are correct. If not, these tests lock in the wrong behavior.

### tests/test_import_agent.py:test_import_agent_propagates_truth_values
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Good targeted test for the golden path: a derived node marked OUT in the source snapshot with a valid justification should flip to IN after propagation. The test constructs a minimal beliefs file and verifies both the propagation count and the final truth value. This is a clean demonstration of the feature's intent.

### Self-Review
**Limitations:** Could not see the full `_propagate` BFS to verify whether it has the same retraction-awareness gap. Did not see `parse_beliefs` to verify how STALE status maps to the claim dict. Did not run the test suite to confirm the 211 tests still pass with these changes. Could not verify whether any other callers of `recompute_all()` exist beyond CLI and import_agent.

### Feature Requests
- Include the full function body for all modified functions (not just diff hunks) in an observations section
- Show callers of newly added methods (`recompute_all`) to verify integration completeness
- Include the test fixture data (`SAMPLE_BELIEFS`) alongside modified test assertions so reviewers can trace test logic without reading the full test file

## gemini [PASS]

### reasons_lib/network.py:recompute_all
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `recompute_all` method implements a robust fixpoint iteration algorithm to update all truth values across the network. The maximum iteration bound of `len(self.nodes) + 1` safely handles worst-case topological updates while preventing infinite loops in the presence of cyclic dependencies.

### reasons_lib/cli.py:cmd_propagate
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Successfully refactors the single-pass manual propagation loop into the new `net.recompute_all()` method. This fixes a latent bug where chained dependencies wouldn't fully propagate in a single run of the `propagate` command. The return type appropriately matches the previous expectation of a list of changed node IDs.

### reasons_lib/cli.py:cmd_import_agent
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Safely logs the newly added `claims_propagated` key if it exists in the import result.

### reasons_lib/import_agent.py:import_agent
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Triggers a full network truth-value recomputation after importing an agent. This correctly ensures that any newly satisfied (or dissatisfied) justifications properly evaluate and update the truth values of both new and existing nodes immediately upon import. Result tracking correctly exposes this to the caller.

### tests/test_import_agent.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Existing tests correctly update their assertions to account for the new behavior (where `gamma-stale` correctly flips to IN upon import, leading to one more affected node upon later retraction). The new `test_import_agent_propagates_truth_values` test cleanly and effectively isolates and proves the new feature.

### Self-Review
**Limitations:** I was unable to inspect the `_compute_truth` and `_log` methods in `network.py` or the saving mechanism in `import_agent.py`, but their correctness is confidently inferred through the context of the diff and test passage.
