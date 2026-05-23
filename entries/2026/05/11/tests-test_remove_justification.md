# File: tests/test_remove_justification.py

**Date:** 2026-05-11
**Time:** 12:47

## Purpose

This file is the test suite for the `remove-justification` feature — the ability to delete a single justification (by index) from a node that has multiple justifications. It validates this operation at three layers of the stack: the in-memory `Network` graph, the persistent `api` module (SQLite-backed), and the `reasons` CLI.

In a TMS, justifications are the *reasons* a node is believed true. Removing one is a structural edit to the belief graph — distinct from retraction (which flips truth value) or node deletion. This test file ensures that removal correctly updates truth values, propagates cascades, cleans up the dependents index, and rejects invalid operations.

## Key Components

### `TestNetworkRemoveJustification` (10 tests)

Tests `Network.remove_justification(node_id, index)` directly on the in-memory graph. Covers:

| Test | What it verifies |
|------|-----------------|
| `test_remove_one_of_two` | Basic removal: correct justification dropped, return value has `removed` and `remaining` |
| `test_remove_causes_out` | Removing the only *valid* justification (leaving one with a retracted antecedent) flips the node OUT |
| `test_remove_propagates_cascade` | Truth-value change cascades to downstream dependents (d depends on c) |
| `test_remove_cleans_dependents` | Antecedent nodes' `dependents` sets are updated — "a" no longer lists "c" after its justification is removed |
| `test_remove_keeps_shared_dependent` | If a node appears in *both* justifications' antecedent lists, removing one justification must *not* remove it from dependents |
| `test_remove_outlist_cleans_dependents` | Outlist nodes (blockers in GATE justifications) also get cleaned from the dependents index |
| `test_error_on_premise` | Premises have no justifications to remove — raises `ValueError` |
| `test_error_on_single_justification` | Refuses to remove the last justification (would orphan the node) — raises `ValueError` |
| `test_error_on_bad_index` | Out-of-range index raises `IndexError` |
| `test_error_on_missing_node` | Nonexistent node raises `KeyError` |

### `TestApiRemoveJustification` (1 test)

Round-trips through the SQLite-backed `api` layer: creates nodes, adds a second justification with `api.add_justification`, removes by index with `api.remove_justification`, and verifies persistence via `api.show_node`.

### `TestCliRemoveJustification` (1 test)

End-to-end: calls `uv run reasons remove-justification c 0` as a subprocess, asserts exit code 0, checks stdout for confirmation text, and verifies the database state via `api.show_node`.

## Patterns

- **Three-layer testing**: Network (unit) → API (integration) → CLI (e2e). Each layer adds persistence/serialization concerns.
- **Shared setup via retraction**: Several tests call `net.retract("b")` to create a node whose justification exists but is unsatisfied. This lets them test truth-value transitions when the *other* justification is removed.
- **`tmp_path` for database isolation**: API and CLI tests use pytest's `tmp_path` fixture to get a throwaway SQLite file per test.
- **Subprocess for CLI tests**: The CLI test shells out to `uv run reasons` rather than invoking the Click app in-process, testing the actual installed entry point.
- **Error boundary tests grouped at the end**: The four `test_error_on_*` methods form a validation contract — documenting every way the call can be rejected.

## Dependencies

**Imports:**
- `pytest` — test framework, `raises` for error assertions
- `reasons_lib.Justification` — the justification dataclass (type, antecedents, outlist, label)
- `reasons_lib.api` — persistence layer (`init_db`, `add_node`, `add_justification`, `remove_justification`, `show_node`)
- `reasons_lib.network.Network` — in-memory TMS graph
- `subprocess` — only in the CLI test class

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical Network-layer test follows this sequence:

1. Create a `Network` instance (empty graph).
2. Add premise nodes (`add_node` with no justifications).
3. Optionally retract a premise to make one justification unsatisfied.
4. Add a derived node with two `Justification` objects.
5. Optionally add a downstream node to test cascade.
6. Call `net.remove_justification(node_id, index)`.
7. Assert on the return dict (`removed`, `remaining`, `old_truth_value`, `new_truth_value`, `changed`) and on the graph state (`nodes[x].truth_value`, `nodes[x].dependents`, `nodes[x].justifications`).

The API-layer test substitutes `api.*` calls for direct `Network` manipulation, and the CLI test substitutes a subprocess call for `api.remove_justification`.

## Invariants

1. **Minimum justification count**: A derived node must retain at least one justification — removing the last is rejected with `ValueError("only one justification")`.
2. **Premises are immutable**: `remove_justification` on a premise raises `ValueError("premise")`.
3. **Dependents index consistency**: After removal, a node must not appear in any antecedent's `dependents` set *unless* that antecedent is still referenced by a remaining justification. This is tested both for inlist (antecedents) and outlist nodes.
4. **Truth-value propagation**: If removal causes a node to go OUT, all downstream dependents must also be re-evaluated (cascade).
5. **Return contract**: The result dict contains `removed` (the deleted justification), `remaining` (count), `old_truth_value`, `new_truth_value`, and `changed` (set of affected nodes).

## Error Handling

All four error paths are tested explicitly:

| Error | Type | Match string | Condition |
|-------|------|-------------|-----------|
| Node not found | `KeyError` | `"not found"` | Node ID doesn't exist in the network |
| Premise node | `ValueError` | `"premise"` | Node has no justifications (it's a premise) |
| Single justification | `ValueError` | `"only one justification"` | Would leave a derived node with zero justifications |
| Bad index | `IndexError` | `"out of range"` | Index >= len(justifications) |

These are all raised by `Network.remove_justification` and propagated unmodified through the API layer. The CLI layer translates them to non-zero exit codes and stderr messages (not tested here).

## Topics to Explore

- [function] `reasons_lib/network.py:remove_justification` — The implementation under test; see how it handles dependents cleanup, truth-value recalculation, and cascade propagation
- [file] `tests/test_add_justification.py` — The complementary operation; compare how add and remove mirror each other's dependents-index maintenance
- [function] `reasons_lib/api.py:remove_justification` — The persistence layer that serializes/deserializes justifications to SQLite and delegates to Network
- [file] `tests/test_outlist.py` — Deeper coverage of outlist/GATE behavior, which `test_remove_outlist_cleans_dependents` touches only lightly
- [general] `dependents-index-consistency` — The dependents index is a derived data structure that must stay in sync across add, remove, retract, and assert operations; understanding its invariants is critical for maintaining the TMS

## Beliefs

- `remove-justification-requires-minimum-two` — `Network.remove_justification` refuses to remove the last justification from a derived node, enforcing a minimum count of one
- `remove-justification-cleans-dependents-index` — After removing a justification, antecedent and outlist nodes' `dependents` sets are updated to remove the target node — but only if the node doesn't appear in another remaining justification's antecedent list
- `remove-justification-cascades-truth-value` — If removing a justification causes a node to flip from IN to OUT, the change propagates to all downstream dependents
- `remove-justification-rejects-premises` — Calling `remove_justification` on a premise node raises `ValueError`, since premises have no justifications to remove

