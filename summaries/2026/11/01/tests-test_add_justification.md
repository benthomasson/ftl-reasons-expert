---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_add_justification.py

# `tests/test_add_justification.py`

## Purpose

This file is the test suite for the **add-justification** operation in the `ftl-reasons` system — a defeasible logic / argumentation engine. It verifies that appending a new justification to an existing node correctly updates truth values, propagates changes through the dependency graph, and persists across save/load cycles.

The file tests this operation at **two architectural layers**:

1. **`TestNetworkAddJustification`** — exercises the in-memory `Network` class directly (unit-level, no I/O).
2. **`TestAPIAddJustification`** — exercises the higher-level `api` module which wraps `Network` with SQLite persistence and a friendlier keyword-argument interface (integration-level).

The responsibility this file owns: ensuring that `add_justification` is a **safe, reversible-in-effect, cascade-aware** mutation on the argumentation graph. It does *not* test the initial node creation, node removal, or the justification *evaluation* logic itself — only the act of appending a new justification and its downstream consequences.

## Key Components

### `TestNetworkAddJustification` (lines 10–113)

| Method | Contract |
|--------|----------|
| `test_add_justification_to_derived_node` | Adding a second SL justification to a node already IN keeps it IN; returns a dict with `node_id`, `old_truth_value`, `new_truth_value`; node's `justifications` list grows to length 2. |
| `test_added_justification_keeps_node_in_after_retract` | A node with two justifications remains IN after one antecedent is retracted (redundancy). |
| `test_add_justification_restores_out_node` | An OUT node transitions to IN when a *new* valid justification is added; `"b"` appears in the `changed` set of the result. |
| `test_add_justification_cascades` | Restoring node `b` also restores dependent `d`; `result["changed"]` includes `"d"`. |
| `test_add_justification_with_outlist` | A justification whose `outlist` contains an IN node is *invalid* (node stays OUT). Retracting the outlist member later activates the justification. |
| `test_add_justification_to_premise` | Premises are not exempt — they can gain justifications as "backup routes." |
| `test_add_justification_nonexistent_node` | Raises `KeyError` with message containing `"not found"`. |
| `test_add_justification_registers_dependents` | Both antecedent nodes and outlist nodes gain the target in their `dependents` set. |

### `TestAPIAddJustification` (lines 116–183)

Uses a `db` fixture (SQLite in `tmp_path`). Tests:

- **SL, CP, and `unless` (outlist) justification types** via keyword args (`sl=`, `cp=`, `unless=`).
- **Validation**: calling `api.add_justification` with no justification keyword raises `ValueError("Must provide...")`.
- **Namespace scoping**: nodes are addressed as `"ns:b"` when a namespace is supplied.
- **Persistence**: after `add_justification`, `api.show_node` reflects the new justification count — confirming the change is written to disk.

### `Justification` dataclass (imported from `reasons`)

Constructed with:
```python
Justification(type="SL", antecedents=["a"], outlist=["enemy"])
```
- `type`: `"SL"` (subsumption link / rule), `"CP"` (conditional premise), or similar.
- `antecedents`: list of node IDs that must be IN.
- `outlist`: list of node IDs that must be OUT (defeaters/enemies).

## Patterns

- **Two-tier testing**: The same logical operation is tested at the raw `Network` level (fast, no disk) and at the `api` level (slower, exercises serialization/deserialization). This catches bugs in the API's argument-parsing layer that pure unit tests would miss.
- **Truth-value state machine**: Tests treat `IN` / `OUT` as the only two states and verify transitions are correct. There is no `UNDECIDED` or `BOTH` state in these tests.
- **Result-dict contract**: `net.add_justification(...)` returns a dict with keys `node_id`, `old_truth_value`, `new_truth_value`, and `changed` (a set/list of all node IDs whose truth value flipped). Tests assert on all of these.
- **`tmp_path` fixture for DB isolation**: Each API test gets a fresh SQLite file, avoiding cross-test contamination.
- **No mocking**: The tests use real `Network` instances and real SQLite. The `db` fixture calls `api.init_db(db_path=...)` — no mocks, no fakes. This is an integration test, not a mock-heavy unit test.

## Dependencies

### Imports (what this file depends on)

```python
from reasons import Justification, api      # public package exports
from reasons.network import Network          # low-level graph engine
```

- `reasons/__init__.py` re-exports `Justification` and `api`.
- `reasons/network.py` provides the `Network` class with `add_node`, `add_justification`, `retract`, and the `.nodes` dict.
- `reasons/api.py` provides the persistence-backed CRUD surface (`add_node`, `add_justification`, `show_node`, `retract_node`, `init_db`).

### What depends on this file (imported_by)

None. This is a **leaf test module**. It is discovered by pytest via its `test_*.py` naming convention and `tests/conftest.py`. No other module imports it.

### Indirect dependencies (via `conftest.py`)

`tests/conftest.py` likely sets up shared fixtures (e.g., resetting global API state, providing `tmp_path` from pytest). The `db` fixture here is local to the class.

## Flow

Each test follows this execution pattern:

```
1. CONSTRUCT   →  Network() or api.init_db(...)
2. SEED        →  add_node(...) for premises and derived nodes
3. (optional)  →  retract(...) to create an OUT state
4. MUTATE      →  add_justification(node_id, Justification(...))
5. ASSERT      →  truth_value, justifications list length,
                  result dict keys, dependents set, cascade targets
```

**Cascading flow** (most important):
```
retract("a")
  → b goes OUT (its only justification's antecedent is gone)
    → d goes OUT (its antecedent b is OUT)

add_justification("b", SL(["c"]))
  → b goes IN (new justification satisfied)
    → d goes IN (cascade: b's dependents re-evaluated)
  → result["changed"] == {"b", "d"}
```

The cascade is driven by the `dependents` registry on each node — when `b`'s truth value changes, the engine iterates `b.dependents` and re-evaluates them.

**Outlist flow**:
```
add_justification("b", SL([], outlist=["enemy"]))
  → enemy is IN → outlist NOT satisfied → b stays OUT

retract("enemy")
  → enemy goes OUT → b is a dependent of enemy → re-evaluate b
  → outlist now satisfied → b goes IN
```

## Invariants

The tests encode these rules that the system must uphold:

1. **Monotonicity of justification count**: Adding a justification never removes an existing one. `len(node.justifications)` only grows.
2. **Truth-value binary**: Nodes are either `"IN"` or `"OUT"` — no third state in these tests.
3. **SL validity rule**: A SL justification is *active* iff all its `antecedents` are IN AND all its `outlist` entries are OUT. A node is IN iff at least one justification is active.
4. **Cascade completeness**: If node X's truth value changes, *all* nodes in `X.dependents` are re-evaluated, and their changes propagate further transitively.
5. **Dependent registration**: When a justification with antecedents `[a, c]` and outlist `[e]` is added to node `b`, the string `"b"` must appear in `a.dependents`, `c.dependents`, *and* `e.dependents`. This is what enables cascade.
6. **Premise parity**: Premises are first-class nodes that can receive justifications — they are not structurally different from derived nodes.
7. **Namespace isolation**: Namespaced nodes are addressed as `"ns:id"` and do not collide with the global namespace.

## Error Handling

| Scenario | Behavior | Test |
|----------|----------|------|
| `add_justification("ghost", ...)` on nonexistent node | Raises `KeyError` with `"not found"` in message | `test_add_justification_nonexistent_node` |
| `api.add_justification("a")` with no `sl`/`cp`/`unless` kwarg | Raises `ValueError` matching `"Must provide"` | `test_add_justification_no_args_raises` |
| Justification with unsatisfied outlist | Does **not** raise; silently leaves node OUT | `test_add_justification_with_outlist` |

There is no test for adding a justification with a *nonexistent antecedent ID* (e.g., `antecedents=["nonexistent"]`). This is a potential gap — the engine may either treat the missing node as perpetually OUT (justification never fires) or raise. The tests don't pin this behavior down.

---

## Topics to Explore

- [function] `reasons/network.py:add_justification` — The core mutation: how it re-evaluates the node, updates `dependents`, and triggers the cascade. Understanding the exact algorithm here is essential for writing new tests or modifying evaluation order.
- [file] `tests/test_remove_justification.py` — The inverse operation. Comparing add vs. remove reveals whether the system treats justifications as an ordered list or a set, and whether removing one can "un-reveal" a latent OUT state.
- [file] `reasons/api.py` — The API layer that translates keyword args (`sl=`, `cp=`, `unless=`) into `Justification` objects and delegates to `Network`. Shows how persistence and validation are layered on top of the in-memory engine.
- [file] `tests/test_dependents_integrity.py` — Likely tests that the `dependents` set is always consistent (no dangling entries, symmetric registration). Understanding this guards against a class of silent cascade bugs.
- [general] `defeasible-logic-semantics` — The IN/OUT truth-value system and outlist (defeater) mechanism is borrowed from defeasible logic programming (Nute, 1987). Reading the README or `reasons/network.py` docstrings will clarify how SL vs. CP justifications differ in priority.

---

## Beliefs

- `add-justification-returns-changed-set` — `Network.add_justification` always returns a dict containing a `changed` key whose value includes every node (including the target) whose truth value flipped during the resulting cascade.
- `node-in-iff-active-justification` — A node's truth value is `"IN"` if and only if at least one of its justifications has all antecedents IN and all outlist entries OUT; otherwise it is `"OUT"`.
- `dependents-registered-on-add` — When a justification is added to node `b` with antecedents `[a]` and outlist `[e]`, the ID `"b"` is inserted into both `a.dependents` and `e.dependents`; this registration is what enables cascade re-evaluation.
- `api-layer-wraps-network-with-sqlite` — `reasons.api` is a persistence-backed facade over `reasons.network.Network`; every mutation goes through SQLite and is verified by a subsequent `show_node` read in these tests.
- `premise-can-receive-justification` — There is no type distinction between "premise" and "derived" nodes at the `Network` level; both are entries in `net.nodes` and both accept `add_justification` calls.
