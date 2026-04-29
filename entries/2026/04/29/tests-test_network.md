# File: tests/test_network.py

**Date:** 2026-04-29
**Time:** 16:54

# `tests/test_network.py` — Network Propagation Test Suite

## Purpose

This is the core test suite for `Network`, the in-memory truth maintenance engine. It validates Doyle's 1979 TMS algorithm: adding nodes with justifications, retracting premises with cascading propagation, restoring retracted nodes, detecting contradictions (nogoods), and producing explanation traces. Every fundamental invariant of the belief network is exercised here — if this file passes, the propagation engine is sound.

## Key Components

### Test Classes (9 total, each covering one behavioral axis)

| Class | What it tests |
|---|---|
| `TestAddNode` | Node creation — premises start IN, derived nodes evaluate their justifications, duplicates are rejected, dependents are wired up |
| `TestRetraction` | Retraction cascades — the core Doyle mechanism. Single node, chain, diamond, alternate justification survival, sticky retraction pinning |
| `TestRestoration` | Re-asserting a retracted node propagates IN back through dependents |
| `TestMultipleAntecedents` | Conjunction semantics — `SL(A, B, C)` requires all three IN |
| `TestNogood` | Contradiction detection — `add_nogood(["a", "d"])` forces at least one OUT, records the nogood, and is a no-op when already satisfied |
| `TestExplain` | Trace generation — `explain()` returns a list of steps walking back from the target to its premises, including failed antecedents for OUT nodes |
| `TestBeliefSet` | `get_belief_set()` returns exactly the set of IN node IDs |
| `TestLog` | Audit trail — `net.log` records `add`, `retract`, and `propagate` events |
| `TestDiamondDependency` | Diamond-shaped dependency graphs (A→B, A→C, B+C→D), including interaction with sticky retraction |
| `TestDanglingDependents` | Graceful handling when a node's `dependents` set references a node that doesn't exist in `net.nodes` |

### Critical Tests

**`test_retract_already_out_pins_retracted`** — Tests "sticky retraction": explicitly retracting a node that's already OUT sets `_retracted` metadata, which prevents it from being resurrected when its antecedent is restored. This is the mechanism that separates "OUT because derivation failed" from "OUT because a human said so." Also verifies that `recompute_all()` respects the pin.

**`test_diamond_with_retracted_intermediate`** — The most complex scenario: in a diamond graph, explicitly retract an intermediate node B, then cycle A through retract/assert. B stays pinned OUT, D stays OUT (missing antecedent), C follows A normally. Then explicitly asserting B clears the pin and D comes back. This is a full round-trip through the sticky retraction lifecycle.

**`test_retract_does_not_cascade_with_alternate_justification`** — Validates that a node with multiple justifications stays IN when only one justification's antecedent is retracted. This is the "alternative support" rule fundamental to non-monotonic reasoning.

## Patterns

1. **Arrange-Act-Assert with fresh `Network()`** — Every test constructs its own network from scratch. No shared fixtures, no database. This makes each test self-contained and deterministic.

2. **Return-value assertions on mutations** — `retract()` and `assert_node()` return the list of changed node IDs. Tests assert both the final truth values *and* the returned change sets, validating the API contract.

3. **Error-path coverage via `pytest.raises`** — Duplicate adds, missing nodes on retract/assert/explain/nogood all raise with specific match strings (`"already exists"`, `"not found"`).

4. **Topological build order** — Tests always add antecedents before dependents, matching the requirement that `add_node` evaluates justifications against existing nodes at insertion time.

## Dependencies

**Imports:**
- `pytest` — test framework
- `reasons_lib.Node`, `Justification`, `Nogood` — data classes (only `Justification` is used directly; `Node` and `Nogood` are imported but exercised indirectly)
- `reasons_lib.network.Network` — the system under test

**Imported by:** Nothing — this is a leaf test module.

**Relationship to production code:** This file tests `Network` in isolation. `Network` is the pure in-memory engine; the persistence layer (`storage.py`) and CLI (`cli.py`) are tested separately. The `conftest.py` may provide shared fixtures but none are used here.

## Flow

Each test follows the same pattern:

1. **Build** — Create a `Network()`, add premise nodes (which start IN), then add derived nodes with `Justification(type="SL", antecedents=[...])` 
2. **Mutate** — Call `retract()`, `assert_node()`, `add_nogood()`, or `explain()`
3. **Assert** — Check `node.truth_value`, the returned change list, `net.nogoods`, `net.log`, or `net.get_belief_set()`

The `Network` internally uses `_propagate()` to walk the dependency graph on retract/assert, flipping truth values and collecting changed nodes. The tests verify this propagation through chains (A→B→C), diamonds (A→B,C→D), and multi-justification nodes.

## Invariants Enforced

- **Premises start IN** with no justifications
- **Derived nodes evaluate immediately** — truth value at insertion depends on current antecedent state
- **Node IDs are unique** — duplicate `add_node` raises `ValueError`
- **Retraction cascades transitively** — if A→B→C and A goes OUT, both B and C go OUT
- **Alternate justifications prevent cascade** — B with SL(A) and SL(C) stays IN when only A is retracted
- **Sticky retraction survives restoration** — `_retracted` metadata prevents resurrection via `assert_node` or `recompute_all`
- **Assert clears sticky retraction** — explicitly asserting a pinned node clears `_retracted` and re-enables normal propagation
- **Retract/assert on missing nodes raises `KeyError`**
- **Retract on already-OUT / assert on already-IN returns `[]`** (idempotent no-ops)
- **Dangling dependents are skipped with a warning**, not a crash

## Error Handling

All error paths use exceptions:
- `ValueError("already exists")` — duplicate node ID on `add_node`
- `KeyError("not found")` — referencing a nonexistent node in `retract`, `assert_node`, `explain`, or `add_nogood`

The one defensive case is `TestDanglingDependents`: when `dependents` contains a reference to a node not in `net.nodes`, propagation logs a warning (`action="warn"`) and continues rather than crashing. This is a robustness guard against index corruption.

## Topics to Explore

- [file] `reasons_lib/network.py` — The production `Network` class this file tests — see `_propagate`, `_evaluate_justification`, and the sticky `_retracted` metadata mechanism
- [function] `reasons_lib/network.py:recompute_all` — Full re-evaluation used in sticky retraction tests; understanding its traversal order is key to understanding why pinned nodes stay OUT
- [file] `tests/test_outlist.py` — Tests for outlist-based non-monotonic justifications (the "unless" clause), which this file does not cover
- [file] `reasons_lib/storage.py` — The persistence layer that serializes `Network` state to SQLite; understanding the boundary between in-memory and persisted state clarifies why these tests need no database
- [general] `doyle-tms-algorithm` — The 1979 paper by Jon Doyle that defines the SL justification semantics, retraction cascades, and nogood resolution this test suite validates

## Beliefs

- `test-network-covers-all-propagation-paths` — `test_network.py` exercises single-node retraction, chain cascades, diamond dependencies, multi-justification survival, sticky retraction, and dangling-dependent handling — the complete set of propagation edge cases
- `retract-returns-changed-set` — `Network.retract()` returns a list of all node IDs whose truth value changed, including the target and all transitively affected dependents
- `sticky-retraction-survives-recompute-all` — A node with `_retracted` metadata stays OUT even when `recompute_all()` re-evaluates the entire network, not just when `assert_node` restores its antecedent
- `network-tests-are-database-free` — Every test in `test_network.py` uses a fresh in-memory `Network()` with no database, fixtures, or shared state
- `add-node-evaluates-justification-at-insertion` — When `add_node` is called with justifications, the node's initial truth value is computed immediately from the current state of its antecedents (tested by `test_add_derived_node_antecedent_out`)

