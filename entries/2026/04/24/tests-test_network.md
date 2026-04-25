# File: tests/test_network.py

**Date:** 2026-04-24
**Time:** 16:52

## `tests/test_network.py` — Explanation

### 1. Purpose

This is the core test suite for the dependency network and propagation engine in `reasons_lib.network.Network`. It validates the fundamental TMS (Truth Maintenance System) operations: adding nodes, retraction cascades, restoration, contradiction handling, and explanation tracing. If this file's tests pass, the foundational belief propagation logic is sound.

### 2. Key Components

**`TestAddNode`** — Verifies node creation semantics:
- A node with no justifications is a **premise** (IN by default).
- A derived node evaluates its justification immediately on creation — if antecedents are OUT, the new node starts OUT.
- Duplicate node IDs raise `ValueError`.
- Adding a derived node registers it in the antecedent's `dependents` set (the reverse pointer that makes propagation possible).

**`TestRetraction`** — The heart of Doyle's TMS. Tests that:
- Retracting a premise sets it OUT and returns the changed set.
- Retraction **cascades** through dependency chains (A→B→C: retract A, both B and C go OUT).
- Retracting an already-OUT node is a no-op (returns `[]`), but still sets `_retracted` metadata to **pin** it — preventing resurrection during later restoration.
- A node with **multiple justifications** survives if any alternative justification remains valid.

**`TestRestoration`** — The complement of retraction:
- `assert_node` sets a node IN and BFS-propagates, restoring any dependents whose justifications become valid again.
- Restoration cascades through chains just like retraction does.
- Asserting an already-IN node is a no-op.

**`TestMultipleAntecedents`** — Validates SL (support-list) conjunction: a justification `SL(A, B, C)` requires **all** antecedents IN. Retracting any one invalidates the derived node; restoring it brings the derived node back.

**`TestNogood`** — Contradiction detection:
- `add_nogood` records a set of mutually exclusive nodes and retracts one to resolve the contradiction.
- If one node is already OUT, no retraction is needed.
- References to nonexistent nodes raise `KeyError`.

**`TestExplain`** — Trace generation for debugging beliefs:
- Premises get a one-step trace (`"reason": "premise"`).
- Derived nodes trace recursively back to their antecedents.
- OUT nodes include `failed_antecedents` identifying what broke the justification.

**`TestBeliefSet`** — `get_belief_set()` returns all node IDs currently IN. Simple projection of network state.

**`TestLog`** — The propagation audit trail records `add`, `retract`, and `propagate` events with targets, enabling post-hoc debugging of cascade behavior.

**`TestDiamondDependency`** — The classic diamond graph (A→B, A→C, B+C→D). Tests:
- Full retract/restore round-trips through the diamond.
- **Sticky retraction**: explicitly retracting an intermediate node (B) pins it OUT even when its antecedent (A) is restored. D stays OUT because B is pinned. Only an explicit `assert_node("b")` clears the pin.
- `recompute_all()` also respects the `_retracted` pin.

**`TestDanglingDependents`** — Defensive handling: if a node's `dependents` set references a nonexistent node, propagation skips it and logs a warning rather than crashing.

### 3. Patterns

- **Arrange-Act-Assert**: Every test builds a small network, performs an operation, and checks resulting truth values and change sets.
- **Incremental complexity**: Tests progress from single-node operations → chains → diamonds → edge cases, mirroring how the propagation algorithm handles increasing graph complexity.
- **Change-set verification**: `retract()` and `assert_node()` return lists of changed node IDs, and tests verify both the network state *and* the returned change set.
- **Class-per-concern grouping**: Each test class isolates one TMS concept (retraction, restoration, nogoods, etc.), making failures easy to localize.

### 4. Dependencies

**Imports:**
- `pytest` — test framework, used for `pytest.raises` assertions.
- `reasons_lib.Node`, `Justification`, `Nogood` — dataclasses representing the TMS primitives.
- `reasons_lib.network.Network` — the system under test.

**Imported by:** Nothing — this is a leaf test file. It's run by pytest.

### 5. Flow

Each test instantiates a fresh `Network()`, adds nodes (premises and derived), then exercises one operation (`retract`, `assert_node`, `add_nogood`, `explain`). The `Network` internally runs BFS propagation on each mutation, so tests can immediately assert the resulting truth values without explicitly calling a propagation step.

The `_retracted` metadata flag is a key flow detail: it acts as a "sticky" marker that prevents BFS restoration from flipping a node back to IN. The `test_retract_already_out_pins_retracted` and `test_diamond_with_retracted_intermediate` tests specifically verify this flow.

### 6. Invariants

- **A premise with no justifications starts IN.** A derived node starts IN only if at least one justification is satisfied.
- **SL justifications are conjunctive**: all antecedents must be IN.
- **Multiple justifications are disjunctive**: a node is IN if *any* justification is valid.
- **Retraction is idempotent**: retracting an OUT node returns `[]` (but may set the `_retracted` pin).
- **Propagation is complete**: after any mutation, all reachable dependents have correct truth values — no explicit propagation call needed.
- **`_retracted` is sticky**: a pinned node stays OUT through `assert_node` on its antecedents *and* through `recompute_all`. Only explicit `assert_node` on the pinned node itself clears it.
- **Dangling dependents don't crash**: propagation skips missing nodes and logs a warning.

### 7. Error Handling

All error cases use `pytest.raises`:
- `ValueError("already exists")` — duplicate node ID on `add_node`.
- `KeyError("not found")` — `retract`, `assert_node`, `explain`, or `add_nogood` referencing a nonexistent node.

No errors are swallowed. The network raises immediately on invalid input, and the tests verify both the exception type and the message pattern.

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class implementation: `_propagate` BFS, `_retracted` pin logic, and `recompute_all` are the code paths these tests exercise.
- [function] `reasons_lib/network.py:_propagate` — The BFS propagation loop that makes retraction cascades and restoration work — the algorithm at the center of every test here.
- [file] `tests/test_outlist.py` — Non-monotonic justifications (outlist): "believe X unless Y" — the other half of SL justifications not covered in this file.
- [file] `tests/test_dialectical.py` — Challenge/defend mechanics build on top of retraction and restoration; understanding how those compose with the base TMS operations.
- [general] `sticky-retraction-semantics` — The `_retracted` metadata pin is a departure from pure Doyle TMS; understanding when and why explicit retraction differs from cascade-induced OUT is critical for correct usage.

## Beliefs

- `premise-default-in` — A node added without justifications is a premise and defaults to truth value IN.
- `sl-conjunction-any-disjunction` — Within a single SL justification, antecedents are conjunctive (all must be IN); across multiple justifications on one node, they are disjunctive (any valid one suffices).
- `retracted-pin-survives-recompute` — A node with `_retracted` metadata stays OUT through both `assert_node` on antecedents and `recompute_all`; only `assert_node` on the node itself clears the pin.
- `propagation-is-immediate` — `retract` and `assert_node` trigger BFS propagation internally; callers never need to invoke propagation separately to get correct truth values.
- `dangling-dependents-log-not-crash` — If a node's dependents set references a nonexistent node, propagation logs a warning with action `"warn"` and continues without raising.

