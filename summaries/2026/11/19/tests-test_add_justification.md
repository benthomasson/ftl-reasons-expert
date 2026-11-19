---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_add_justification.py

# `tests/test_add_justification.py`

## Purpose

This file is a pytest module that defines the behavioral contract for “adding a justification” to an existing reason node. It covers two layers:

1. **Core network behavior** via `reasons.network.Network.add_justification`
2. **Persistent API behavior** via `reasons.api.add_justification`

It verifies that adding an alternative justification can:

- increment a node’s justification list
- restore an `OUT` node to `IN`
- keep a node `IN` after one supporting antecedent is retracted
- cascade truth-value changes to dependents
- respect `outlist` defeat semantics
- register dependency edges for future propagation
- persist through the SQLite-backed API layer

The file does not implement production logic. It is an executable specification for the `add_justification` feature.

## Key Components

### `TestNetworkAddJustification`

Tests the in-memory `Network` behavior directly.

| Test | Contract exercised |
|---|---|
| `test_add_justification_to_derived_node` | Appends a second `Justification`; result contains `node_id`, `old_truth_value`, `new_truth_value`; node remains `IN`; justification count becomes `2`. |
| `test_added_justification_keeps_node_in_after_retract` | Multiple justifications are alternatives: after retracting one antecedent, the node stays `IN` because the added justification still supports it. |
| `test_add_justification_restores_out_node` | If retracting `a` makes `b` `OUT`, adding a valid justification from `c` changes `b` back to `IN`; result reports `old_truth_value == "OUT"` and `new_truth_value == "IN"`; `"b"` appears in `result["changed"]`. |
| `test_add_justification_cascades` | Restoring `b` also restores dependent `d`; `"d"` appears in `result["changed"]`. |
| `test_add_justification_with_outlist` | A justification with `outlist=["enemy"]` is invalid while `enemy` is `IN`; after `net.retract("enemy")`, `b` becomes `IN` automatically. This proves `outlist` nodes are registered as dependencies and retraction triggers reevaluation. |
| `test_add_justification_to_premise` | Premises are not immutable: adding an `SL` justification to an existing premise node is allowed, and the justification list grows to `1`. |
| `test_add_justification_nonexistent_node` | Adding to a missing node raises `KeyError` with message matching `"not found"`. |
| `test_add_justification_registers_dependents` | Adding a justification registers the target node under the dependents of both antecedent nodes and outlist nodes. |

Key contract from these tests:

- `Network.add_justification(node_id, justification)` mutates the network and returns a dict with at least:
  - `node_id`
  - `old_truth_value`
  - `new_truth_value`
  - `changed`
- Truth values are strings, at least `"IN"` and `"OUT"`.
- Justifications are appended, not replaced.
- Multiple justifications behave as alternative support paths.
- Dependency edges are updated for antecedents and outlist nodes.

### `TestAPIAddJustification`

Tests the persistent API wrapper around the same feature.

The `db` fixture:

```python
@pytest.fixture
def db(self, tmp_path):
    db_path = str(tmp_path / "reasons.db")
    api.init_db(db_path=db_path)
    return db_path
```

creates a fresh SQLite database per test using pytest’s `tmp_path`.

| Test | Contract exercised |
|---|---|
| `test_add_sl_justification` | `api.add_justification("b", sl="c", db_path=db)` appends an `SL` justification; `show_node("b")` reports `2` justifications. |
| `test_add_cp_justification` | `api.add_justification("b", cp="a", db_path=db)` appends a `CP` justification. The test asserts `justifications[1]["type"] == "CP"`, so ordering matters: new justifications are appended. |
| `test_add_unless_only_justification` | The API accepts `unless="enemy"` as the sole justification argument. Note: the target node already has a valid `sl="a"` justification and `a` remains `IN`, so this test primarily verifies argument acceptance, not full outlist validity behavior. |
| `test_add_justification_no_args_raises` | Calling the API with no `sl`, `cp`, or `unless` raises `ValueError` matching `"Must provide"`. |
| `test_add_justification_with_namespace` | Namespaced nodes are supported; adding to `b` in namespace `"ns"` returns `node_id == "ns:b"`. |
| `test_add_justification_restores_out_node` | After `api.retract_node("a")`, `b` is `OUT`; adding `sl="c"` restores it to `IN`, with result reporting old/new truth values. |
| `test_add_justification_persists` | A justification added through the API is visible through `show_node`, demonstrating SQLite persistence. |

The API-level contract is:

- `api.add_justification(...)` accepts `sl`, `cp`, `unless`, `namespace`, and `db_path`.
- At least one of `sl`, `cp`, or `unless` must be provided.
- The result includes the same truth-value metadata as the network layer.
- Namespaced results use qualified node IDs such as `"ns:b"`.
- API calls use explicit `db_path=...`; there is no global database state in these tests.

## Patterns

- **Layered testing**: the file separates in-memory graph semantics (`Network`) from SQLite-backed API semantics (`api`).
- **Fixture isolation**: each API test gets a fresh temporary database through `tmp_path`.
- **Arrange–Act–Assert**: tests seed nodes, call `add_justification`, then assert node truth values, justification counts, result metadata, and dependents.
- **Executable specification**: the tests define the expected result dict shape and truth-maintenance behavior.
- **Incremental truth maintenance testing**: multiple tests exercise alternative justifications, retraction, propagation, and `outlist` defeat semantics.
- **Real dependencies, no mocks**: tests use the real `Network` class and real SQLite database rather than stubs.

## Dependencies

This test file depends on:

- `pytest` for test cases, fixtures, and `pytest.raises`
- `reasons.Justification`
- `reasons.api`
- `reasons.network.Network`

The production modules under test are primarily:

- `reasons/network.py`: `Network.add_justification`, `Network.add_node`, `Network.retract`, node/dependent state
- `reasons/api.py`: `init_db`, `add_node`, `add_justification`, `show_node`, `retract_node`

No production code imports this test file. It is discovered by pytest and depends on the test package setup in `tests/__init__.py`.

Repository note: the repo contains both `reasons/` and `reasons_lib/` with overlapping module names. This test imports from `reasons`, not `reasons_lib`, so it only protects the top-level `reasons` implementation.

## Flow

### Network test flow

1. Construct an empty `Network`.
2. Add premise/derived nodes with `add_node`.
3. Seed some nodes with `Justification(type="SL", antecedents=[...])`.
4. Optionally call `net.retract(...)` to move nodes to `OUT`.
5. Call `net.add_justification(node_id, Justification(...))`.
6. Assert:
   - returned `node_id`
   - old and new truth values
   - `changed` contains the target and/or cascaded dependents
   - `net.nodes[node_id].justifications` length
   - dependency registration on antecedent/outlist nodes

### API test flow

1. `db` fixture creates a temporary SQLite path and calls `api.init_db(db_path=...)`.
2. Test seeds nodes with `api.add_node(..., db_path=db)`.
3. Test optionally retracts a node with `api.retract_node`.
4. Test calls `api.add_justification(..., db_path=db)` with one of `sl`, `cp`, or `unless`.
5. Test asserts result metadata.
6. Test calls `api.show_node(..., db_path=db)` to verify persisted justifications and truth values.

Data transformation summary:

- API arguments such as `sl="c"`, `cp="a"`, and `unless="enemy"` are translated into a `Justification`.
- The justification is appended to the target node’s justification list.
- Truth values are recomputed and propagated through dependents.
- The API persists the mutation in SQLite.
- The result dict reports the target node and truth-value transition.

## Invariants

- A target node must exist before a justification can be added; otherwise `KeyError` is raised with `"not found"`.
- Adding a justification appends to the existing list; it does not replace existing justifications.
- Multiple valid justifications are alternatives: retracting one antecedent does not make a node `OUT` if another valid justification remains.
- Adding a valid justification to an `OUT` node can restore it to `IN`.
- Restoring a node can cascade to its dependents, and affected dependents are included in `result["changed"]`.
- An `outlist` node must be `OUT` for that justification to be valid.
- Antecedent and outlist nodes must receive the target as a dependent so future truth changes trigger reevaluation.
- Premises can have justifications added to them.
- API `add_justification` requires at least one of `sl`, `cp`, or `unless`.
- In namespaced mode, returned node IDs are qualified, e.g. `"ns:b"`.
- Added justifications are appended in order; `test_add_cp_justification` asserts the new justification appears at index `1`.

## Error Handling

- Missing target node: `Network.add_justification("ghost", ...)` raises `KeyError` with a message matching `"not found"`.
- API call with no justification arguments: `api.add_justification("a", db_path=db)` raises `ValueError` with a message matching `"Must provide"`.
- `pytest.raises(..., match=...)` uses regex matching, so the production error messages must contain those substrings.
- The tests do not swallow exceptions; they assert that errors propagate out of the network/API layers.
- The tests do not explicitly cover missing antecedents, missing outlist nodes, duplicate justifications, or invalid `CP` arguments.

## Topics to Explore

- [file] `reasons/network.py` — Implements `Network.add_justification`, dependent tracking, truth-value propagation, and the `changed` result contract.
- [function] `reasons/network.py:Network.add_justification` — The core mutation and truth-maintenance routine exercised by every network test in this file.
- [file] `reasons/api.py` — Implements `api.add_justification`, argument validation, namespace handling, and SQLite persistence.
- [function] `reasons/api.py:add_justification` — Shows how `sl`, `cp`, and `unless` arguments map into `Justification` objects and how results are serialized.
- [general] `truth-maintenance-semantics` — Explore `IN`/`OUT` propagation, `SL` vs `CP` justifications, `outlist` defeat behavior, and dependent cascade rules.

## Beliefs

- `network-add-justification-returns-truth-transition-metadata` — `Network.add_justification` returns a dict containing `node_id`, `old_truth_value`, `new_truth_value`, and `changed`.
- `api-add-justification-requires-a-justification-argument` — `api.add_justification` raises `ValueError` matching `"Must provide"` when none of `sl`, `cp`, or `unless` is supplied.
- `add-justification-registers-antecedent-and-outlist-dependents` — Adding a justification registers the target node in the dependents of each antecedent and outlist node.
- `multiple-justifications-are-alternative-support-paths` — A node with two valid justifications remains `IN` after one supporting antecedent is retracted.
- `namespaced-add-justification-returns-qualified-node-id` — With `namespace="ns"`, `api.add_justification` returns `node_id == "ns:b"`.
