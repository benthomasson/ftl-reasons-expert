# File: tests/test_import_agent.py

**Date:** 2026-05-08
**Time:** 14:11

# `tests/test_import_agent.py` — Multi-Agent Belief Import Tests

## Purpose

This file is the test suite for the `import_agent` feature — the mechanism by which one TMS (Truth Maintenance System) database ingests an entire external belief network from another agent, namespaced so that multiple agents' beliefs can coexist without collisions. It validates the full lifecycle: import, namespace isolation, dependency remapping, retraction cascades, restoration, deduplication, and format support (both Markdown and JSON).

This is a critical test file because multi-agent federation is one of the more complex features in `ftl-reasons`. Bugs here — particularly around retraction semantics — have real consequences (see the issue #16 regression tests).

## Key Components

### Test Data Constants

- **`SAMPLE_BELIEFS`** — A minimal Markdown belief export containing three beliefs: an IN observation (`alpha-fact`), an IN derived belief that depends on it (`beta-depends-alpha`), and a STALE observation (`gamma-stale`). This covers the three key import cases: premise, derived with dependency, and non-IN status.

- **`SAMPLE_NOGOODS`** — A Markdown nogood (contradiction record) referencing two beliefs. Used by `test_import_agent_nogoods`.

### Fixtures

- **`db`** — Creates a fresh SQLite database in a temp directory via `api.init_db()`. Every test gets an isolated database.

- **`beliefs_file`** — Writes `SAMPLE_BELIEFS` and `SAMPLE_NOGOODS` to temp files, returns the path to the beliefs file. The nogood file is expected to be discovered by convention (same directory, `nogoods.md`).

### Test Functions (grouped by concern)

**Basic import mechanics:**
- `test_import_agent_basic` — Verifies the return dict: agent name, namespace prefix, active node name, premise creation flag, import/retract counts.
- `test_import_agent_creates_premise` — The `{agent}:active` sentinel node is created as an IN premise with `role=agent_premise` metadata.
- `test_import_agent_namespaces_beliefs` — Every imported belief gets prefixed (`test-agent:alpha-fact`) and carries `original_id` metadata.
- `test_import_agent_remaps_dependencies` — Derived beliefs' `Depends on:` references are rewritten to namespaced IDs. The `active` node is *not* added as an antecedent; instead, `inactive` appears in the outlist as a kill switch.

**Retraction cascades (the core TMS behavior):**
- `test_import_agent_retract_premise_cascades` — `what_if_retract` on `active` reports all IN beliefs would be affected.
- `test_import_agent_retract_premise_actually_cascades` — Actually retracting `active` flips all imported beliefs to OUT.
- `test_import_agent_restore_premise_cascades` — Re-asserting `active` after retraction brings beliefs back to IN.

**Deduplication:**
- `test_import_agent_skip_duplicates` — Second import of the same agent/beliefs returns `claims_imported=0, claims_skipped=3`.

**Filtering:**
- `test_import_agent_only_in` — `only_in=True` skips STALE/OUT beliefs entirely; they don't exist in the database at all.

**Multi-agent isolation:**
- `test_import_multiple_agents` — Two agents import the same beliefs. Each gets its own namespace. Retracting one agent's `active` node doesn't affect the other's beliefs.

**OUT-status preservation:**
- `test_import_agent_preserves_out_status` — A belief marked OUT in the source stays OUT after import, even if its justification antecedents are satisfied.
- `test_out_with_satisfied_justifications_stays_out` — JSON variant: an OUT node whose SL justification *would* be satisfied is not resurrected to IN.

**Outlist / non-monotonic reasoning:**
- `test_import_agent_preserves_outlist` — `Unless: blocker` lines are namespaced and preserved. If the blocker is IN, the gated belief stays OUT.
- `test_import_agent_supersession_preserved` — Supersession pattern (v1 OUT because v2 IN) survives import with correct outlist wiring.

**JSON import path:**
- `test_import_agent_json` — Imports from `network.json` format, validates full justification structure including antecedents, outlist, and the `inactive` kill switch.
- `test_import_agent_json_outlist_blocks` — When a blocker node is IN in the JSON source, gated beliefs correctly stay OUT.

**Regression tests (issue #16):**
- `test_retracted_belief_survives_propagate` — After retraction, running `recompute_all()` must not resurrect retracted beliefs. This was the core bug: the `active` premise as antecedent provided an always-valid fallback justification.
- `test_retracted_json_belief_survives_propagate` — Same regression test via JSON import path.

**Metadata:**
- `test_import_agent_registers_repo` — `api.list_repos()` includes the agent with the source directory path.
- `test_import_agent_nogoods` — Nogood records from the source are imported (count = 1).

## Patterns

1. **Fixture isolation** — Every test gets a fresh `tmp_path` database and fresh belief files. No shared mutable state between tests.

2. **Return-dict assertions** — `import_agent` returns a summary dict rather than raising; tests assert against its keys. This is the project's convention for operations that produce structured results.

3. **Round-trip testing** — Several tests import, mutate (retract/assert), then verify. The `recompute_all` regression tests go further: import → retract → recompute → verify truth values haven't changed.

4. **Inline test data** — Complex tests (JSON import, outlist, supersession) define their own belief data inline rather than sharing fixtures. This keeps each test self-documenting at the cost of some repetition.

5. **Late imports** — `json`, `Storage`, and `Path` are imported inside test function bodies rather than at module level. This is a minor style choice — these are only needed by specific tests.

## Dependencies

**Imports:**
- `reasons_lib.api` — The public API surface. Every test goes through `api.import_agent`, `api.show_node`, `api.retract_node`, `api.assert_node`, `api.what_if_retract`, `api.get_status`, `api.list_repos`, `api.init_db`.
- `reasons_lib.storage.Storage` — Used directly in the two `recompute_all` regression tests to trigger propagation outside the normal API.
- `pytest` — Fixtures and `pytest.raises`.

**Imported by:** Nothing — this is a test module.

**Implicitly exercises:** `reasons_lib.import_agent` (the implementation module), `reasons_lib.network` (TMS engine), `reasons_lib.storage` (SQLite persistence).

## Flow

A typical test follows this sequence:

1. **Setup** — `db` fixture creates a fresh SQLite database; `beliefs_file` or inline data provides the source beliefs.
2. **Import** — `api.import_agent(agent_name, source_path, db_path=db)` parses the source, creates the `{agent}:active` premise, namespaces all beliefs, remaps dependencies, and writes everything to the database.
3. **Verify initial state** — `api.show_node()` checks truth values, metadata, and justification structure.
4. **Mutate** (optional) — `api.retract_node()` or `api.assert_node()` triggers TMS propagation.
5. **Verify post-mutation state** — Truth values reflect correct cascade behavior.

The regression tests add a step between 4 and 5: manually loading the network via `Storage`, calling `recompute_all()`, and saving — simulating what happens during a full re-propagation.

## Invariants

1. **Namespace isolation** — Every imported belief ID is prefixed with `{agent}:`. Two agents importing the same source never collide.
2. **Kill switch** — Every imported belief has `{agent}:inactive` in its outlist. When `inactive` goes IN (i.e., `active` goes OUT), all beliefs cascade to OUT.
3. **Dependency remapping** — `Depends on:` and `Unless:` references in the source are rewritten to namespaced IDs. No dangling references to un-namespaced IDs.
4. **OUT preservation** — If a belief is OUT or STALE in the source, it stays OUT after import. The TMS engine does not resurrect it based on satisfied justifications.
5. **Retraction stickiness** — Once a belief is retracted, `recompute_all()` must not bring it back. This is the invariant that issue #16 violated.
6. **Idempotent re-import** — Importing the same agent twice produces `claims_imported=0, claims_skipped=N`.

## Error Handling

- `test_import_agent_only_in` uses `pytest.raises(KeyError)` to verify that excluded beliefs don't exist at all — `show_node` raises `KeyError` for missing nodes.
- No other error paths are explicitly tested. The tests assume `import_agent` succeeds or the assertion on return values catches problems.

---

## Topics to Explore

- [file] `reasons_lib/import_agent.py` — The implementation this test suite exercises; the namespace remapping, kill-switch wiring, and OUT-preservation logic live here
- [function] `reasons_lib/network.py:recompute_all` — The propagation engine that the regression tests verify against; understanding how it evaluates justifications explains why the `active`-as-antecedent bug was so subtle
- [function] `reasons_lib/api.py:import_agent` — The public API entry point; shows how Markdown vs JSON format detection works and how the return dict is assembled
- [general] `tms-outlist-semantics` — How outlist (non-monotonic "unless") clauses interact with the kill switch (`inactive` node) to provide both per-belief gating and whole-agent retraction
- [file] `reasons_lib/storage.py` — The SQLite persistence layer; understanding how justifications are serialized and loaded explains the round-trip fidelity these tests verify

## Beliefs

- `import-agent-namespaces-all-beliefs` — Every belief imported by `import_agent` is prefixed with `{agent}:` and no un-namespaced references survive in justifications
- `import-agent-inactive-outlist-kill-switch` — Every imported belief includes `{agent}:inactive` in its outlist, enabling whole-agent retraction by flipping a single node
- `import-agent-out-beliefs-not-resurrected` — Beliefs marked OUT or STALE in the source are imported as OUT and never resurrected by `recompute_all`, even when their justification antecedents are satisfied
- `import-agent-idempotent-reimport` — Re-importing the same agent and beliefs skips all existing claims (`claims_imported=0`) without creating duplicates
- `import-agent-active-not-antecedent` — The `{agent}:active` premise does not appear as an antecedent in imported beliefs' justifications (fix for issue #16); only the `inactive` outlist provides the agent-level toggle

