---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_outlist.py

# Explanation of `tests/test_outlist.py`

## Purpose

`tests/test_outlist.py` is a pytest behavioral test suite for **non-monotonic SL justifications with outlists**.

It verifies the default-logic semantics:

> “Believe X unless Y is believed.”

In this implementation, an `SL` justification can have:

- `antecedents`: the “inlist” — nodes that must be `IN`
- `outlist`: nodes that must be `OUT`

The file owns tests for the core outlist behavior, cascading truth-value changes, multiple competing justifications, explanation output, persistence through SQLite, and the public API’s `unless=` support.

Production code is not defined here; this file defines the contract the `reasons` package must satisfy.

---

## Key Components

### Test Classes

#### `TestOutlistBasic`

Tests fundamental truth-value behavior:

- `test_outlist_valid_when_out`: `X` with outlist `["y"]` is `IN` when `y` is retracted/`OUT`.
- `test_outlist_invalid_when_in`: `X` becomes `OUT` when any outlist node is `IN`.
- `test_outlist_with_inlist`: verifies both inlist and outlist must be satisfied: `a` is `IN`, `y` is `OUT`, so `X` is `IN`.
- `test_outlist_with_inlist_both_fail`: inlist failure alone is enough to make `X` `OUT`, even if the outlist condition is satisfied.
- `test_outlist_node_not_in_network`: a nonexistent outlist node is treated as `OUT`, so it does not block the justification.

#### `TestOutlistCascading`

Tests dynamic propagation after `assert_node` / `retract`:

- `test_assert_outlist_node_retracts_dependent`: asserting `y` invalidates `x` and `x` appears in the returned `changed` set.
- `test_retract_outlist_node_restores_dependent`: retracting `y` restores `x` to `IN`.
- `test_outlist_cascade_through_chain`: invalidation propagates through dependency edges: `y` asserted → `x` `OUT` → `z` `OUT`.
- `test_multiple_outlist_nodes`: `x` requires all outlist nodes to be `OUT`; asserting just one outlist node invalidates `x`.

#### `TestOutlistWithMultipleJustifications`

Tests OR semantics across justifications:

- `test_alternate_justification_survives_outlist`: `x` has two `SL` justifications. When the first is invalidated by asserting `y`, the second keeps `x` `IN`. Retracting its sole antecedent then makes `x` `OUT`.

#### `TestOutlistExplain`

Tests `Network.explain("x")` output:

- Valid case: `steps[0]["truth_value"] == "IN"` and `steps[0]["outlist"] == ["y"]`.
- Invalid case: `steps[0]["truth_value"] == "OUT"` and `steps[0]["violated_outlist"] == ["y"]`.

#### `TestOutlistPersistence`

Tests SQLite round-trip through `Storage`:

- Saves a network with an `SL` justification containing `outlist=["y"]` and `label="default unless Y"`.
- Loads it back and verifies:
  - `outlist` is preserved.
  - `label` is preserved.
  - truth value is preserved.
  - outlist propagation still works after loading.

#### `TestOutlistAPI`

Tests the public `reasons.api` layer:

- `api.add_node(..., unless="y", ...)` creates a node that is `IN` while `y` is `OUT`.
- Asserting `y` later makes `x` `OUT`, observable through `api.get_status(...)`.
- `api.add_node(..., sl="a", unless="y", ...)` supports both inlist and outlist.
- `unless` without `sl` still produces `result["type"] == "SL"`.

### Production Contracts Exercised

- `Justification(type="SL", antecedents=[...], outlist=[...], label=...)`
- `Network.add_node(...)`
- `Network.retract(...)`
- `Network.assert_node(...)`
- `net.nodes["id"].truth_value`
- `Network.explain(...)`
- `Storage.save(net)`, `Storage.load()`, `Storage.close()`
- `api.init_db(...)`
- `api.add_node(..., sl=..., unless=..., db_path=...)`
- `api.retract_node(...)`
- `api.assert_node(...)`
- `api.get_status(...)`
- pytest’s `tmp_path` fixture

---

## Patterns

- **Class-based pytest organization**: Tests are grouped by behavioral concern.
- **Arrange-Act-Assert**: Each test builds a small network, performs an operation, then asserts truth values or explanation output.
- **Fresh state per test**: Each test constructs its own `Network`, avoiding shared mutable state.
- **Real components instead of mocks**: The tests use actual `Network`, `Storage`, and `api` implementations rather than mocking internals.
- **Temporary database isolation**: Persistence and API tests use pytest’s `tmp_path` fixture to create isolated SQLite databases.
- **Behavioral contract testing**: Most tests exercise public methods and assert observable truth-value changes rather than inspecting internal implementation details.
- **Cascade testing**: Assertions check both final truth values and, in some cases, the returned `changed` set from `assert_node` / `retract`.

---

## Dependencies

### Imports

- `pytest`
  - Imported at module level, though not used directly in the shown tests.
  - `tmp_path` fixture injection is provided by pytest.
- `from reasons import Justification`
- `from reasons.network import Network`
- `from reasons.storage import Storage`
- `from reasons import api`

The tests target the `reasons` package, not the parallel-looking `reasons_lib` package.

### Depends On

- `reasons.network.Network`: truth maintenance, outlist propagation, explanation generation.
- `reasons.storage.Storage`: persistence of justifications and node truth values.
- `reasons.api`: higher-level DB-backed API operations.
- `reasons.Justification`: representation of inlist/outlist justifications.

### Imported By

- No production code should import this test module.
- It is collected and executed by pytest/CI.

---

## Flow

A typical test follows this flow:

1. Create a fresh `Network`.
2. Add supporting nodes such as `y`, `a`, or `z`.
3. Set initial truth values using `add_node`, `retract`, or `assert_node`.
4. Add a dependent node `x` with one or more `Justification` objects:
   - `antecedents` act as the inlist.
   - `outlist` acts as the “must be OUT” list.
5. Trigger a state change, usually by asserting or retracting an outlist node.
6. Assert the resulting `truth_value` of affected nodes.
7. For persistence/API tests:
   - Create a temporary SQLite DB.
   - Save or mutate through `Storage` / `api`.
   - Reload or query status.
   - Assert that outlist state and propagation behavior survived.

Examples:

- In `test_outlist_valid_when_out`, `y` is retracted first, then `x` is added with `outlist=["y"]`; `x` evaluates to `IN`.
- In `test_assert_outlist_node_retracts_dependent`, asserting `y` changes `x` from `IN` to `OUT`.
- In `TestOutlistPersistence`, saving/loading preserves both the serialized justification and the evaluated truth value.

---

## Invariants

The tests establish these rules:

- An `SL` justification is valid only when:
  - Every node in `antecedents` / inlist is `IN`.
  - Every node in `outlist` is `OUT`.
- A missing outlist node is treated as `OUT`; absence is not an error.
- Any single outlist node being `IN` invalidates that justification.
- A node is `IN` if at least one of its justifications is valid.
- A node is `OUT` if none of its justifications are valid.
- `assert_node` and `retract` propagate changes to dependent nodes.
- The `changed` set returned by `assert_node` / `retract` includes affected dependent node IDs.
- `Network.explain` reports:
  - `outlist` for valid outlist justifications.
  - `violated_outlist` for invalidated outlist justifications.
- `Storage.save` / `Storage.load` preserve `outlist` and `label`.
- Outlist propagation still works after a persisted network is loaded.
- `api.add_node(unless="y")` creates an `SL`-type node whose justification has `y` in its outlist.
- `api.add_node(sl="a", unless="y")` combines antecedent `a` and outlist `y`.

---

## Error Handling

This test file does not assert exceptions or use `pytest.raises`.

Notable error-related behavior:

- A nonexistent outlist node is not treated as an error; it is treated as `OUT`.
- Failed truth-value expectations surface as normal pytest `AssertionError`s.
- Persistence tests assume `Storage.save` / `Storage.load` succeed for valid temporary SQLite paths.
- API tests assume `api.init_db`, `api.add_node`, `api.retract_node`, and `api.assert_node` operate successfully with a valid `db_path`.

There is no explicit testing of malformed justifications, invalid node IDs in `antecedents`, or database failure modes.

---

## Topics to Explore

- [file] `reasons/network.py` — Implements `add_node`, `assert_node`, `retract`, outlist propagation, and `explain`.
- [file] `reasons/storage.py` — Handles SQLite serialization of justifications, including `outlist` and `label`.
- [file] `reasons/api.py` — Maps `unless=` and `sl=` API arguments onto `SL` justifications.
- [function] `reasons/network.py:Network.assert_node` — Returns the `changed` set and triggers dependent-node recomputation.
- [general] `doyle-outlist-semantics` — The underlying default-logic truth-maintenance model behind inlist/outlist justifications.

---

## Beliefs

- `outlist-invalidates-when-node-in` — An `SL` justification with an outlist is invalid if any outlist node is `IN`.
- `missing-outlist-nodes-are-out` — A nonexistent outlist node is treated as `OUT` and does not block the justification.
- `node-truth-is-disjunction-of-justifications` — A node is `IN` when at least one of its justifications is valid.
- `assert-node-propagates-outlist-changes` — Asserting an outlist node recomputes dependent nodes and includes them in the returned `changed` set.
- `storage-round-trip-preserves-outlist` — `Storage.save` and `Storage.load` preserve `Justification.outlist` and `label`.
