# File: tests/test_import_agent.py

**Date:** 2026-04-24
**Time:** 16:46

# `tests/test_import_agent.py` — Multi-Agent Belief Import Tests

## Purpose

This file is the test suite for the **agent import** feature — the ability to ingest an external agent's beliefs (from markdown or JSON) into the shared reason-maintenance network as a namespaced, independently controllable unit. It validates that importing an agent creates the right dependency structure so that an entire agent's belief set can be activated or deactivated with a single premise toggle, while keeping agents isolated from each other.

This is the test coverage for `reasons_lib/import_agent.py` and the `api.import_agent()` entry point.

## Key Components

### Test Fixtures

- **`SAMPLE_BELIEFS`** — A markdown string representing three beliefs: an IN observation, an IN derived belief with a dependency, and a STALE observation. This is the canonical test input exercised by most tests.
- **`SAMPLE_NOGOODS`** — A markdown string with one nogood (contradiction record) linking two beliefs.
- **`db` fixture** — Creates a fresh SQLite database per test via `api.init_db()` in a temporary directory.
- **`beliefs_file` fixture** — Writes both `SAMPLE_BELIEFS` and `SAMPLE_NOGOODS` to disk, returns the path to the beliefs file. The nogoods file is co-located and discovered by convention.

### Test Functions (grouped by concern)

**Basic import mechanics:**
- `test_import_agent_basic` — Verifies the return dict: agent name, prefix, active node ID, premise creation flag, counts of imported and retracted claims.
- `test_import_agent_creates_premise` — The `{agent}:active` node is IN and carries agent metadata.
- `test_import_agent_namespaces_beliefs` — Every imported belief gets the `{agent}:` prefix; original ID is preserved in metadata.
- `test_import_agent_remaps_dependencies` — Dependency references in justifications are rewritten from `alpha-fact` to `test-agent:alpha-fact`. The `{agent}:inactive` node appears in the outlist as a kill switch.

**Cascade behavior (the core value proposition):**
- `test_import_agent_retract_premise_cascades` — `what_if_retract` on the active premise reports all IN beliefs would be affected.
- `test_import_agent_retract_premise_actually_cascades` — Actually retracting `{agent}:active` flips all agent beliefs to OUT.
- `test_import_agent_restore_premise_cascades` — Re-asserting the active premise brings beliefs back to IN.

**Idempotency and filtering:**
- `test_import_agent_skip_duplicates` — Second import of the same agent returns `claims_imported=0`, `claims_skipped=3`.
- `test_import_agent_only_in` — The `only_in=True` flag skips STALE/OUT beliefs entirely.

**Multi-agent isolation:**
- `test_import_multiple_agents` — Two agents share a database; retracting one doesn't affect the other. Verifies node count arithmetic (2 active + 2 inactive + 6 beliefs = 10).

**Truth-value preservation:**
- `test_import_agent_preserves_out_status` — A belief marked OUT in the source stays OUT after import (not resurrected by propagation).
- `test_import_agent_preserves_outlist` — Non-monotonic `Unless:` relationships are namespaced and functional. An IN blocker correctly suppresses a gated belief.
- `test_import_agent_supersession_preserved` — Superseded beliefs (OUT via outlist in source) are imported as bare premises with no justification, staying OUT.

**JSON import path:**
- `test_import_agent_json` — Full justification structure (antecedents + outlist) survives JSON import with correct namespacing.
- `test_import_agent_json_outlist_blocks` — An IN blocker in JSON input correctly gates a dependent belief to OUT.

**Regression tests (issue #16):**
- `test_retracted_belief_survives_propagate` — After retraction, running `recompute_all()` must not resurrect retracted beliefs. This was the core bug: the active premise used to be an antecedent on every belief, providing a fallback that defeated per-belief retraction.
- `test_retracted_json_belief_survives_propagate` — Same regression test via the JSON import path.

**Nogoods:**
- `test_import_agent_nogoods` — Verifies that contradiction records are imported and counted.

## Patterns

1. **Fixture-per-resource**: `db` and `beliefs_file` are independent fixtures composed by pytest's dependency injection. Each test gets a fresh database — no cross-test contamination.

2. **Round-trip testing**: Import → query → mutate → query. Tests don't just check import succeeded; they verify the imported structure behaves correctly under propagation, retraction, and restoration.

3. **Inline test data**: Beliefs markdown and JSON are defined as constants or built inline rather than read from fixture files. This makes each test self-contained and readable without cross-referencing.

4. **Regression-anchored tests**: `test_retracted_belief_survives_propagate` explicitly references issue #16 and describes the exact bug it guards against. This is the most important pattern in the file — it documents *why* the active premise must not be an antecedent of imported beliefs.

5. **Kill-switch architecture**: The `{agent}:inactive` node in the outlist of every justification means that asserting `{agent}:inactive` would also disable all agent beliefs — a second path to deactivation beyond retracting `{agent}:active`.

## Dependencies

**Imports:**
- `reasons_lib.api` — The public functional API; every test goes through this layer.
- `reasons_lib.storage.Storage` — Used directly only in the two regression tests to call `recompute_all()` at the network level, bypassing the API to simulate what the `propagate` CLI command does.
- `pytest` — Test framework and fixtures.
- `json` — Building JSON import payloads inline.

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test follows this sequence:

1. `api.init_db()` creates a fresh SQLite database.
2. Markdown or JSON belief data is written to a temp file.
3. `api.import_agent(agent_name, file_path, db_path=db)` parses the file, namespaces all node IDs, creates the `{agent}:active` and `{agent}:inactive` control premises, wires up justifications with remapped dependencies, and runs propagation.
4. Tests query individual nodes via `api.show_node()` to verify truth values, justification structure, and metadata.
5. Mutation tests call `api.retract_node()` or `api.assert_node()` and re-query to verify cascade behavior.

## Invariants

- **Namespace isolation**: Every node imported by agent `X` has the ID prefix `X:`. No test ever sees a node without this prefix after import.
- **Active premise controls agent liveness**: Retracting `{agent}:active` must cascade OUT to all agent beliefs. Re-asserting it must restore them.
- **Agent independence**: Retracting agent A's active premise never changes agent B's truth values.
- **Idempotent re-import**: Importing the same agent twice creates zero new nodes.
- **OUT preservation**: Beliefs marked OUT in the source must remain OUT after import — propagation must not resurrect them.
- **Retraction survives recompute**: Once a belief is retracted, `recompute_all()` must not flip it back to IN. This is the issue #16 invariant.

## Error Handling

- `test_import_agent_only_in` uses `pytest.raises(KeyError)` to verify that excluded beliefs are truly absent from the database, not just marked OUT.
- No other tests check error paths — this file is focused on the happy path and the cascade invariants. Error handling for malformed input would be tested elsewhere (or is missing).

---

## Topics to Explore

- [file] `reasons_lib/import_agent.py` — The implementation this file tests; contains the namespacing, justification remapping, and kill-switch wiring logic
- [function] `reasons_lib/network.py:recompute_all` — The propagation engine that the regression tests verify doesn't resurrect retracted beliefs
- [file] `reasons_lib/import_beliefs.py` — The markdown parser that `import_agent` delegates to for parsing `beliefs.md` format
- [file] `tests/test_sync_agent.py` — Likely tests the incremental update path (re-sync) as opposed to first-time import
- [general] `issue-16-active-premise-antecedent-bug` — The architectural decision to remove the active premise from per-belief antecedent lists and use only the outlist kill switch for agent-level control

## Beliefs

- `agent-import-creates-two-control-nodes` — Every `import_agent` call creates exactly two control nodes: `{agent}:active` (IN premise) and `{agent}:inactive` (OUT premise used in outlist kill switches)
- `namespace-prefix-is-agent-colon` — All imported node IDs are prefixed with `{agent}:` and the original ID is preserved in `metadata["original_id"]`
- `active-premise-not-in-antecedents` — After the issue #16 fix, `{agent}:active` does NOT appear in the antecedent list of imported beliefs' justifications — only `{agent}:inactive` appears in the outlist
- `idempotent-reimport-skips-all` — Re-importing the same agent with the same beliefs results in `claims_imported=0` and `claims_skipped=N`
- `out-beliefs-imported-as-bare-premises` — Beliefs that are OUT in the source file are imported as bare premises (empty justification list) and immediately retracted, preventing propagation from resurrecting them

