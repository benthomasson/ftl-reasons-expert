# File: tests/test_dangling_dependents.py

**Date:** 2026-04-24
**Time:** 16:55

# `tests/test_dangling_dependents.py`

## Purpose

This file is the regression test suite for **issue #22** — a bug where the `_propagate` method in `Network` would crash with a `KeyError` when a node's `dependents` set contained an ID that didn't correspond to any node in the network. These "dangling" or "ghost" references can arise from data corruption, incomplete cleanup after node deletion, or deserialization edge cases. The fix added a guard that skips nonexistent dependents and emits a warning log entry instead of crashing.

The file owns the contract: **propagation is resilient to dangling dependent references** — it warns, skips, and continues.

## Key Components

### `TestDanglingDependentNoCrash` (4 tests)
The smoke tests. Each one manually injects a bogus ID (e.g., `"nonexistent"`, `"ghost1"`) into a node's `dependents` set, then triggers propagation via `retract()` or `assert_node()`. The assertion is implicit: if no exception is raised, the test passes. Covers single dangles, multiple dangles, and dangles coexisting with valid dependents.

### `TestDanglingDependentWarningLogs` (5 tests)
Verifies the **observability** contract. After propagation encounters a dangling ref, a warning entry must appear in `net.log` with:
- `action == "warn"`
- `target` set to the ghost node ID
- `value` containing the string `"dangling"` and the parent node's ID
- a `timestamp` field

### `TestDanglingDependentPropagationContinues` (4 tests)
Verifies that dangling refs don't **interrupt** the BFS cascade. Tests increasingly complex topologies — single parent-child, chain (A→B→C with ghost on B), and diamond (A→B, A→C, B+C→D with ghost on A). In every case, valid dependents still get their truth values updated correctly.

### `TestDanglingDependentEdgeCases` (5 tests)
Corner cases:
- **Same ghost in multiple nodes**: warns for each encounter.
- **Ghost not in changed list**: the return value of `retract()` must never include a dangling ID.
- **`challenge()` path**: challenges trigger `_propagate` internally — dangles handled there too.
- **`add_justification()` path**: another indirect `_propagate` caller.
- **Ghost doesn't pollute visited set** (`test_dangling_does_not_enter_visited`): if a ghost ID later becomes a real node, it must not be skipped by a stale visited-set entry. This is the subtlest test — it guards against a specific implementation mistake where the guard might `visited.add(ghost_id)` before skipping.

## Patterns

**Fault injection via internal mutation**: Every test injects the fault by directly mutating `net.nodes["x"].dependents.add("ghost")`. This bypasses the public API (which would never create a dangling ref) to simulate data corruption. This is the right approach — it tests the guard, not the happy path.

**Implicit-pass tests**: The `NoCrash` class uses no `assert` statements. The test passes if no exception is thrown. This is a standard pattern for "must not crash" regression tests.

**Log-as-observable-side-effect**: Warning behavior is tested by inspecting `net.log`, a list of dicts maintained by `Network`. This avoids mocking Python's `logging` module and keeps tests deterministic.

**Topology escalation**: The `PropagationContinues` class escalates from simple (one edge) to complex (diamond), ensuring the fix works at every level of the BFS traversal.

## Dependencies

**Imports**:
- `reasons_lib.Node`, `reasons_lib.Justification` — the core dataclasses. `Node` is imported but unused (likely kept for readability or future use).
- `reasons_lib.network.Network` — the class under test.

**Imported by**: Nothing — this is a leaf test module.

## Flow

Every test follows the same pattern:
1. Create a `Network` instance.
2. Add nodes (premises and/or derived nodes with `Justification` objects).
3. Inject one or more ghost IDs into a node's `dependents` set.
4. Trigger propagation (`retract`, `assert_node`, `challenge`, or `add_justification`).
5. Assert one of: no crash, correct warning logs, correct truth values on valid nodes, or correct return value.

The critical moment is inside `_propagate`, which does a BFS over dependents. When it dequeues a dependent ID and looks it up in `self.nodes`, the guard must catch the missing key, log a warning, and `continue` the loop.

## Invariants

1. **Dangling refs never crash propagation** — `_propagate` must handle `KeyError` gracefully on any code path that triggers it.
2. **One warning per dangling encounter** — each ghost hit produces exactly one log entry.
3. **Warning entries have a stable schema**: `{"action": "warn", "target": <ghost_id>, "value": <message containing "dangling" and parent_id>, "timestamp": ...}`.
4. **Dangling IDs never appear in the `changed` return set** — callers of `retract`/`assert_node` must not see ghost IDs.
5. **Dangling IDs never enter the visited set** — a ghost that later becomes a real node must propagate normally.

## Error Handling

The file doesn't test error *raising* — it tests error *suppression*. The entire point is that `_propagate` catches what would be a `KeyError`, converts it into a structured warning log entry, and continues. No exception reaches the caller.

---

## Topics to Explore

- [function] `reasons_lib/network.py:_propagate` — The actual guard implementation: how it detects dangling refs, logs warnings, and continues the BFS loop
- [file] `tests/test_dependents_integrity.py` — Likely tests the complementary concern: ensuring dependents are correctly maintained under normal operations
- [function] `reasons_lib/network.py:retract` — Entry point that triggers `_propagate`; understanding its return value contract (`changed` set) matters for interpreting these tests
- [general] `doyle-tms-propagation` — Doyle's 1979 BFS propagation algorithm and why dependent tracking is separate from justification tracking
- [file] `reasons_lib/__init__.py` — The `Node` and `Justification` dataclasses, particularly the `dependents` field type and how it's populated during normal `add_node`

## Beliefs

- `dangling-guard-is-continue-not-raise` — When `_propagate` encounters a dependent ID not in `self.nodes`, it logs a warning and continues the BFS loop rather than raising or silently skipping
- `warning-log-schema-stable` — Dangling-dependent warnings use the dict schema `{action: "warn", target: <ghost_id>, value: <str>, timestamp: <str>}` and tests enforce all four fields
- `dangling-ids-excluded-from-changed` — The set returned by `retract()` and `assert_node()` never contains IDs that don't correspond to real nodes in the network
- `dangling-ids-excluded-from-visited` — The propagation visited set does not include dangling IDs, so a formerly-dangling ID that becomes a real node will propagate correctly on subsequent operations

