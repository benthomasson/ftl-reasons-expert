---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_defeat_justification.py

# `tests/test_defeat_justification.py`

## Purpose

This file is an integration-level test suite for the **justification-defeat mechanism** in the `reasons` belief network. It exercises `reasons.api.defeat_justification`, which lets you "disable" a specific derivation (justification) of a derived belief by creating a *defeater* node and wiring it into that justification's `outlist`. It verifies that:

- Defeat toggles the affected belief from `IN` (believed) to `OUT` (disbelieved).
- The effect cascades through downstream dependents.
- Node `IN`/`OUT` state is recomputed correctly in the presence of multiple justifications.
- Defeaters survive export and can be retracted to restore the belief.
- The API raises the expected errors for bad inputs.

It is the behavioral spec that `reasons/api.py`'s defeat path (and the underlying storage/justification engine) must satisfy.

## Key Components

### Test functions

Each test is a self-contained scenario using pytest's `tmp_path` to get a fresh SQLite DB per run.

| Test | What it locks down |
|---|---|
| `test_defeat_justification_basic` | Full happy path: adding premises + a derived belief, asserting it's `IN`, defeating justification `0`, asserting the belief flips to `OUT`, and inspecting the defeater node's metadata. |
| `test_defeat_justification_added_to_outlist` | The defeater ID is appended to `justifications[0]["outlist"]` — this is the mechanism by which defeat is encoded. |
| `test_defeat_restoration_on_retract` | `api.retract_node(defeater_id)` restores the defeated belief and reports it in `retract_result["changed"]`. |
| `test_defeat_cascade_to_dependents` | `b1` defeated ⇒ `b1` and its dependent `b2` both flip to `OUT`, both listed in `result["changed"]`. |
| `test_defeat_multiple_justifications` | OR-semantics: with two justifications, defeating only one leaves the node `IN`; defeating the second flips it to `OUT`. |
| `test_defeat_custom_id` | The `defeater_id` keyword overrides the auto-generated defeater name. |
| `test_defeat_errors` | `KeyError` (missing node), `ValueError` ("no justifications"), `IndexError` ("out of range"). |
| `test_defeater_types` | The four recognized types `invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by` round-trip into `metadata.defeater_type`. |
| `test_defeater_in_export` | `api.export_network` preserves both the defeater node and the outlist edge. |
| `test_premise_defeat` | Premises (nodes without justifications) cannot be defeated — `ValueError`. |

### Contract of `api.defeat_justification`

Inferred from call sites and assertions:

```python
api.defeat_justification(
    node_id: str,
    justification_index: int,
    reason: str,
    defeater_type: str = <default>,     # e.g. "over-generalizes"
    defeater_id: str | None = None,     # default -> "{defeater_type}-{node_id}-j{idx}"
    db_path: str,
) -> {
    "node_id": str,
    "justification_index": int,
    "defeater_type": str,
    "defeater_id": str,
    "changed": list[str],   # node IDs whose truth_value changed
}
```

Default defeater naming is `"{defeater_type}-{node_id}-j{index}"` — proven by `over-generalizes-belief-j0` in `test_defeat_justification_basic`.

### Node shape consumed by tests

From `api.show_node(...)`:

```python
{
    "truth_value": "IN" | "OUT",
    "justifications": [ { "outlist": [<defeater_id>, ...] }, ... ],
    "metadata": {
        "defeater_type": str,           # on defeater nodes
        "defeats_node": str,            # on defeater nodes
        "defeats_justification": int,   # on defeater nodes
    },
}
```

From `api.export_network(...)`: `{"nodes": {node_id: <node record>, ...}}`.

### Other API touch-points

- `api.init_db(db_path)` — create schema.
- `api.add_node(node_id, label, sl="p1,p2", db_path=...)` — `sl` is a comma-separated support list; a node with `sl` is a *derived* belief, without `sl` it is a premise.
- `api.add_justification(node_id, sl=..., db_path=...)` — add an *additional* justification to an existing node.
- `api.retract_node(node_id, db_path=...)` — remove a node and re-propagate.
- `api.export_network(db_path=...)` — dump the whole network.

## Patterns

- **Truth Maintenance System (TMS) with explicit outlists.** The design mirrors Doyle/de Kleer TMS: a justification has premises (support) plus an `outlist` of defeaters; a node is `IN` iff it has at least one *valid* justification (all premises `IN`, no live defeater in `outlist`). Defeat is not deletion — it is *annotation via outlist*, so the record is reversible.
- **Defeaters are first-class nodes.** Rather than a boolean flag, defeat creates a real node in the graph with its own `truth_value`. This makes defeat an ordinary graph object that participates in retraction, export, and future defeats.
- **Deterministic defeater IDs.** The default naming `{type}-{node}-j{idx}` gives idempotent, reproducible identifiers (useful for export diffing and deduplication).
- **Fresh-DB fixture isolation.** Every test uses `tmp_path` + `init_db`, so tests never share state and can be run in any order/parallel.
- **Nested `pytest.raises(..., match=...)`** for precise error-contract assertions — the `match` strings (`"not found"`, `"no justifications"`, `"out of range"`) are part of the public contract.
- **Behavior-over-implementation tests.** Assertions are on `truth_value` and `changed` lists, not on SQL rows — implementation of storage could change without breaking the suite.

## Dependencies

**Imports**
- `pytest` — fixtures (`tmp_path`), `pytest.raises`.
- `reasons.api` — the entire public API under test. This test file knows nothing about storage internals.

**Implicit dependencies (must exist to satisfy the tests)**
- A persistence layer (`reasons/storage.py` or `pg.py`) that stores nodes, justifications, outlists, and metadata.
- A propagation engine that recomputes `truth_value` for the defeated node and its transitive dependents, returning the list of `changed` IDs.
- An exporter (`export_network`) that serializes nodes + outlists.

**Downstream consumers**
- This file is one of several defeat-related tests: `test_classify_defeaters.py`, `test_defeat_with_scope.py`, `test_add_justification.py`, `test_remove_justification.py`, `test_backtracking.py`, `test_dialectical.py`. Changes to `defeat_justification`'s contract will affect all of them.

## Flow

Each test follows the same skeleton:

1. `db = tmp_path / "test.db"; api.init_db(str(db))` — fresh DB.
2. `api.add_node(...)` x N — build the graph (premises first, derived beliefs with `sl=` after).
3. *(optional)* `api.add_justification(...)` — attach an additional derivation path.
4. `api.defeat_justification(node, idx, reason, ...)` — the operation under test.
5. `api.show_node(...)` / `api.export_network(...)` — read back state.
6. `assert` on `result["changed"]`, `truth_value`, `outlist` contents, metadata, or raised exceptions.

Two flows are worth calling out specifically:

- **Cascade** (`test_defeat_cascade_to_dependents`): `p → b1 → b2`. Defeating `b1`'s sole justification must re-evaluate `b1` (goes `OUT`) and then `b2` (its premise `b1` is `OUT`, so `b2` goes `OUT`). Both must appear in `result["changed"]`.
- **Multi-justification** (`test_defeat_multiple_justifications`): after defeating `j0`, the engine must re-evaluate `b` and find `j1` still valid ⇒ `b` stays `IN`. After defeating `j1`, both are out ⇒ `b` flips `OUT`.

## Invariants

1. **Node truth = OR over justifications (or premise).** A node is `IN` iff it is a premise (no justification requirements) or has ≥1 justification whose premises are all `IN` and whose `outlist` contains no `IN` defeater. `test_defeat_multiple_justifications` and `test_premise_defeat` encode this.
2. **Defeat is recorded, not destructive.** The only observable change to the belief's own record is that `defeater_id` is appended to `justifications[idx]["outlist"]`; the justification itself remains present (`test_defeat_justification_added_to_outlist`).
3. **Defeater nodes are `IN` on creation.** `show_node(defeater_id)["truth_value"] == "IN"` immediately after the call (`test_defeat_justification_basic`, `test_defeat_custom_id`).
4. **Defeater metadata contract.** Every defeater node must carry `defeater_type`, `defeats_node`, `defeats_justification` (`test_defeat_justification_basic`, `test_defeater_in_export`).
5. **`changed` is the exhaustive reachable-closure delta.** Every node whose `truth_value` flipped as a result of the operation must be present in `result["changed"]`; nodes that did not flip should not be there (implied by cascade test asserting both `b1` and `b2`).
6. **Default ID determinism.** Omitted `defeater_id` ⇒ `"{defeater_type}-{node_id}-j{index}"` (`test_defeat_justification_basic`).
7. **Premises cannot carry justifications**, therefore cannot be the target of defeat (`test_premise_defeat`).
8. **Export round-trips.** `export_network` must serialize the defeater node and the outlist edge losslessly (`test_defeater_in_export`).

## Error Handling

Three exception contracts are asserted, all via `pytest.raises(..., match=...)`:

| Condition | Exception | `match` substring |
|---|---|---|
| Target node does not exist | `KeyError` | `"not found"` |
| Target node exists but has zero justifications (premise) | `ValueError` | `"no justifications"` |
| `justification_index` beyond `len(justifications)-1` | `IndexError` | `"out of range"` |

All three are raised synchronously by `defeat_justification`; there is no silent fallback and no partial write — the tests don't propose a "best-effort" mode. The choice of exception types is meaningful:
- `KeyError` for the missing scope key (Pythonic for "lookup miss").
- `IndexError` for the index — matches list semantics, so any Python developer intuits it.
- `ValueError` for the semantic violation (defeating a premise makes no sense conceptually).

The `match=` patterns are part of the contract — if the messages change, these tests break, so downstream maintainers should treat those substrings as stable.

## Topics to Explore

- [function] `reasons/api.py:defeat_justification` — The implementation that produces the defeater node, updates the outlist, and triggers propagation. Compare against the sibling `defeat_with_scope` if it exists.
- [file] `tests/test_defeat_with_scope.py` — Likely the same behavior under a `scope` parameter; clarifies whether defeater metadata includes a scope key beyond `defeater_type`/`defeats_node`/`defeats_justification`.
- [file] `tests/test_classify_defeaters.py` — Suggests a classification layer on top of the four defeater types (`invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by`).
- [file] `reasons/storage.py` — The persistence layer; will reveal how `outlist` is stored (JSON column? join table?) and how `changed` is computed in SQL vs Python.
- [general] `truth-maintenance-system` — Background on Doyle/de Kleer TMS semantics — the `IN`/`OUT` truth values and `outlist` vocabulary used throughout these tests are direct borrowings.

## Beliefs

- `defeat-is-implemented-via-outlist` — Defeating a justification works by appending the defeater node ID to that justification's `outlist`, not by deleting the justification.
- `default-defeater-id-follows-pattern` — When `defeater_id` is omitted, the generated ID is `"{defeater_type}-{node_id}-j{justification_index}"`.
- `defeater-types-are-four` — The known `defeater_type` values exercised are `invalid-inference`, `over-generalizes`, `duplicate-of`, and `superseded-by`.
- `premises-cannot-be-defeated` — Calling `defeat_justification` on a node with no justifications raises `ValueError("... no justifications ...")`.
- `defeat-cascades-to-dependents` — Defeating a justification re-propagates and marks every transitive dependent that loses support as `OUT`, all listed in `result["changed"]`.
- `multiple-justifications-or-semantics` — A node with N justifications stays `IN` while at least one justification remains valid, and flips `OUT` only after the last valid justification is defeated.
