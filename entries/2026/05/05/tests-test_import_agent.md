# File: tests/test_import_agent.py

**Date:** 2026-05-05
**Time:** 15:26

# `tests/test_import_agent.py` — Multi-Agent Belief Import

## Purpose

This file is the test suite for the **import-agent** feature: the ability to ingest an external agent's belief network into the local TMS database as a namespaced, isolated sub-network with a single "kill switch" premise. It validates the full lifecycle — import, namespace isolation, dependency remapping, retraction cascades, restore, deduplication, and multi-agent coexistence.

It is one of the more architecturally significant test files because it exercises the federation model: how independent belief systems compose inside a single TMS without interfering with each other.

## Key Components

### Test Data Constants

- **`SAMPLE_BELIEFS`** — A markdown-formatted belief export containing three beliefs: an IN observation (`alpha-fact`), an IN derived belief with a dependency (`beta-depends-alpha`), and a STALE observation (`gamma-stale`). This is the canonical fixture for most tests.
- **`SAMPLE_NOGOODS`** — A markdown nogood record referencing a contradiction between `alpha-fact` and `gamma-stale`.

### Fixtures

- **`db(tmp_path)`** — Creates a fresh SQLite database via `api.init_db()` in a temporary directory. Every test gets an isolated database.
- **`beliefs_file(tmp_path)`** — Writes `SAMPLE_BELIEFS` to `beliefs.md` and `SAMPLE_NOGOODS` to `nogoods.md` in a temp directory, returning the path to the beliefs file.

### Test Functions (grouped by concern)

**Basic import mechanics:**
- `test_import_agent_basic` — Verifies the return dict: agent name, namespace prefix, active node name, premise creation flag, import/retract counts.
- `test_import_agent_creates_premise` — The agent gets a `<agent>:active` premise node marked IN with `role=agent_premise` metadata.
- `test_import_agent_namespaces_beliefs` — Every imported belief ID is prefixed with `<agent>:`, and metadata preserves the `original_id`.

**Dependency remapping:**
- `test_import_agent_remaps_dependencies` — Imported justifications remap antecedents from bare IDs to namespaced IDs (`alpha-fact` → `test-agent:alpha-fact`). The active premise is **not** added as an antecedent. An `<agent>:inactive` node is added to the outlist as a kill switch.

**Retraction cascade (the core TMS property):**
- `test_import_agent_retract_premise_cascades` — `what_if_retract` on `<agent>:active` shows all 3 IN beliefs would be affected.
- `test_import_agent_retract_premise_actually_cascades` — Actually retracting the active premise flips all imported beliefs to OUT (4 total changes including the premise itself).
- `test_import_agent_restore_premise_cascades` — Re-asserting the active premise restores the cascade — beliefs come back IN.

**Idempotency:**
- `test_import_agent_skip_duplicates` — A second import of the same agent with the same beliefs imports 0 new claims and reports `claims_skipped=3`.

**Filtering:**
- `test_import_agent_only_in` — The `only_in=True` flag skips STALE/OUT beliefs entirely; they don't exist in the database at all (raises `KeyError`).

**Multi-agent isolation:**
- `test_import_multiple_agents` — Two agents importing the same beliefs get fully independent namespaces. Retracting one agent's premise doesn't affect the other. Verifies total node count: 2 active + 2 inactive + 6 beliefs = 10.

**Truth value preservation:**
- `test_import_agent_preserves_out_status` — A belief marked OUT in the source snapshot stays OUT after import; the recompute doesn't resurrect it.
- `test_import_agent_preserves_outlist` — Unless/outlist relationships are namespaced and functional: an IN blocker keeps the gated belief OUT.
- `test_import_agent_supersession_preserved` — Supersession pattern: v1 is OUT in source with an `Unless: v2` relationship. v1 is imported as a bare premise (no justification) and stays OUT.

**JSON import path:**
- `test_import_agent_json` — Full justification structure (SL type, antecedents, outlist) survives JSON import with correct namespacing and truth value computation.
- `test_import_agent_json_outlist_blocks` — When an outlist node is IN in the JSON source, the gated belief correctly evaluates to OUT.

**Regression tests (issue #16):**
- `test_retracted_belief_survives_propagate` — The critical regression: after retracting an imported belief, running `recompute_all()` must not resurrect it. This was the core bug — the active premise as an antecedent provided an always-valid fallback that defeated per-belief retraction.
- `test_retracted_json_belief_survives_propagate` — Same regression test via the JSON import path.

**Metadata:**
- `test_import_agent_registers_repo` — The agent's source directory is registered in the repos list via `api.list_repos()`.
- `test_import_agent_nogoods` — Nogoods from the companion `nogoods.md` file are imported alongside beliefs.

## Patterns

1. **Fixture-per-concern**: `db` handles database lifecycle, `beliefs_file` handles test data. They compose independently via pytest's dependency injection.

2. **Inline test data for variants**: Tests that need non-standard belief structures (outlist, supersession, JSON) define their own belief text inline rather than parameterizing the shared fixture. This makes each test self-contained and readable.

3. **Assert-the-return-then-verify-state**: Most tests first check the return dict from `import_agent`, then independently verify the database state via `show_node`. This catches bugs where the return value lies about what actually happened.

4. **Regression-as-narrative**: The two `survives_propagate` tests include docstrings explaining the specific bug they prevent (issue #16). They import → retract → recompute → assert, which mirrors the exact sequence that triggered the original defect.

5. **Kill switch pattern**: The agent architecture uses a sentinel premise (`<agent>:active`) whose retraction cascades through all imported beliefs. An `<agent>:inactive` node in every outlist provides the complementary kill switch mechanism.

## Dependencies

**Imports:**
- `reasons_lib.api` — The primary API surface. All import, show, retract, assert, status, and repo operations go through this module.
- `reasons_lib.storage.Storage` — Used directly only in the two regression tests to call `recompute_all()`, bypassing the API to simulate what the propagate command does.
- `pytest` — Test framework and fixtures.
- `json`, `os`, `tempfile`, `pathlib.Path` — Standard library utilities for test data setup.

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test follows this sequence:

1. **Setup**: `db` fixture creates an empty SQLite database; `beliefs_file` writes markdown to disk.
2. **Import**: `api.import_agent(agent_name, file_path, db_path=db)` parses the belief file, namespaces all IDs, creates the active/inactive sentinel nodes, remaps justification dependencies, and writes everything to the database.
3. **Verify import results**: Check the return dict for counts and flags.
4. **Verify database state**: `api.show_node(namespaced_id, db_path=db)` retrieves individual nodes to check truth values, metadata, and justification structure.
5. **Exercise lifecycle**: Retract the active premise, verify cascade. Re-assert, verify restoration. Import again, verify deduplication.

## Invariants

1. **Namespace isolation**: Every imported belief ID must be prefixed with `<agent>:`. No bare IDs should exist in the database from an import operation.
2. **Active premise is IN**: After a successful import, `<agent>:active` must be IN with `role=agent_premise` metadata.
3. **Kill switch completeness**: Every imported belief's justification must include `<agent>:inactive` in its outlist, ensuring a single retraction point.
4. **Retraction stability**: Once a belief is retracted, `recompute_all()` must not resurrect it. The justification structure must not provide alternative paths to IN.
5. **Idempotent re-import**: Importing the same agent and beliefs twice must produce `claims_imported=0` and `created_premise=False`.
6. **Source truth value fidelity**: OUT/STALE beliefs in the source must remain OUT after import — the TMS must not "improve" their truth value.
7. **Multi-agent independence**: Retracting agent A's premise must not change any truth values in agent B's namespace.

## Error Handling

The tests primarily verify correct behavior rather than error paths. The one explicit error-path test is `test_import_agent_only_in`, which asserts that `api.show_node` raises `KeyError` when looking up a belief that was excluded by the `only_in=True` filter. No tests cover malformed input, missing files, or database corruption — those would belong in the API or storage test suites.

## Topics to Explore

- [file] `reasons_lib/import_agent.py` — The implementation that these tests exercise; understanding the namespacing and justification remapping logic is essential context
- [function] `reasons_lib/api.py:import_agent` — The public API entry point; how it dispatches between markdown and JSON parsing, and how it constructs the active/inactive sentinel nodes
- [function] `reasons_lib/network.py:recompute_all` — The propagation engine; the two regression tests exist specifically because this function previously resurrected retracted beliefs
- [file] `tests/test_import_beliefs.py` — The non-agent import path; comparing the two reveals what the agent layer adds (namespacing, kill switch, isolation)
- [general] `kill-switch-outlist-pattern` — How the `<agent>:inactive` outlist entry interacts with TMS justification semantics to provide single-point retraction

## Beliefs

- `import-agent-namespaces-all-ids` — Every belief imported by `import_agent` has its ID prefixed with `<agent>:`, and justification antecedents/outlists are remapped to use the same prefix
- `import-agent-active-premise-not-antecedent` — The `<agent>:active` sentinel is not added as an antecedent to imported justifications; the kill switch works exclusively through the `<agent>:inactive` outlist entry
- `import-agent-idempotent-reimport` — Calling `import_agent` twice with the same agent name and beliefs produces zero new imports and does not create a duplicate premise
- `retracted-beliefs-stable-across-recompute` — After a belief is retracted via `retract_node`, calling `recompute_all()` must not flip it back to IN — this was a regression (issue #16) caused by the active premise providing an always-satisfied fallback justification
- `multi-agent-retraction-isolated` — Retracting one agent's active premise cascades only through that agent's namespace; other agents' beliefs are unaffected

