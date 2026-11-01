---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_dangling_dependents.py

# `tests/test_dangling_dependents.py`

## 1. Purpose

This file is a **regression test suite** for issue #22 in the `ftl-reasons` project. It validates that the `Network._propagate` method gracefully handles a specific corruption state: a node's `dependents` set containing references to node IDs that don't exist in `net.nodes`. Instead of crashing with a `KeyError`, the system must log a structured warning and continue propagating to valid dependents.

The file owns the **contract** that "dangling dependent references are non-fatal, observable, and non-blocking." Every test here is a guard against a developer accidentally reintroducing a bare `net.nodes[dep_id]` lookup inside the propagation loop.

## 2. Key Components

### `TestDanglingDependentNoCrash`
Four minimal smoke tests proving the system doesn't raise `KeyError`:
- `test_retract_with_single_dangling_dependent` — adds a ghost to `dependents`, calls `retract`.
- `test_assert_with_single_dangling_dependent` — same but via the `assert_node` (re-assertion) path.
- `test_multiple_dangling_refs_on_one_node` — three ghosts at once.
- `test_dangling_ref_alongside_valid_dependent` — one ghost + one real dependent (`b`); asserts `b` still transitions to `"OUT"`.

### `TestDanglingDependentWarningLogs`
Verifies the **log contract** — each warning entry in `net.log` must be a dict with:
- `action == "warn"`
- `target` = the ghost node ID
- `value` containing the substring `"dangling"` and the parent node's ID
- `timestamp` key present

Tests:
- `test_single_warning_logged` — exact count (1) and content checks.
- `test_warning_includes_parent_node_id` — the `value` string must name `"a"`.
- `test_multiple_dangling_produce_multiple_warnings` — 2 ghosts → exactly 2 entries, distinct targets.
- `test_warning_on_assert_path` — confirms `assert_node` also routes through the same guard.
- `test_warning_has_timestamp` — structural completeness of the log entry.

### `TestDanglingDependentPropagationContinues`
Proves the propagation **workload** is unaffected:
- `test_valid_dependent_still_retracted` — ghost doesn't block `b` from cascading OUT.
- `test_valid_dependent_still_restored` — ghost doesn't block `b` from cascading IN.
- `test_chain_with_dangling_in_middle` — A→B→C chain; ghost on B; C still reaches OUT.
- `test_diamond_with_dangling` — A→{B,C}→D diamond; ghost on A; D reaches OUT; ghost absent from `changed`.

### `TestDanglingDependentEdgeCases`
- `test_same_dangling_in_multiple_nodes` — same ghost ID on two different nodes' `dependents` sets; at least one warning for it.
- `test_dangling_not_in_changed_list` — the returned `changed` set/list excludes ghosts.
- `test_challenge_with_dangling` — `challenge()` also triggers `_propagate`; guard fires.
- `test_add_justification_with_dangling` — `add_justification` → `retract` path; guard fires.
- `test_dangling_does_not_enter_visited` — the critical invariant that ghost IDs are **not** added to the BFS/DFS `visited` set. The test then materializes the ghost as a real node and confirms it propagates normally (OUT → IN).

## 3. Patterns

| Pattern | Where |
|---|---|
| **State-injection testing** | Tests deliberately corrupt internal state (`net.nodes["a"].dependents.add("ghost")`) to simulate corruption that production code is expected to tolerate. This is a *chaos-monkey* idiom applied at unit scale. |
| **Log-assertion contract** | Rather than monkey-patching `logging`, the suite inspects `net.log` (an in-memory event log on the `Network` instance). This couples tests to the log-entry schema (`action`, `target`, `value`, `timestamp`). |
| **Path-coverage matrix** | The same guard is exercised through every public entry point that calls `_propagate`: `retract`, `assert_node`, `challenge`, `add_justification`. This is a deliberate "all-callers" sweep. |
| **Graph-topology progression** | Tests escalate from single-node → chain → diamond, mirroring how a developer would reason about BFS/DFS propagation. |
| **Negative-assertion pairs** | Every "X still works" test is paired with "ghost NOT in changed" / "ghost NOT in visited," pinning both the positive and negative contract. |

## 4. Dependencies

**Imports (this file → project):**
- `from reasons import Justification, Node` — domain model classes. `Justification` is used to create SL (single-link) justifications that tie antecedents to derived nodes. `Node` is imported but only used implicitly (via `net.nodes`).
- `from reasons.network import Network` — the system under test. All tests construct a fresh `Network()` per test.

**Depends-on (project → this file):**
- No production code imports this file. It is a leaf test module.
- `tests/conftest.py` (sibling) likely provides shared fixtures (e.g., a `network` fixture), though this file constructs its own `Network()` instances inline, suggesting it intentionally avoids shared state.

**Upstream (what this file guards):**
- `reasons/network.py:_propagate` (private method) — the code that iterates `node.dependents` and looks up `net.nodes[dep_id]`.
- `reasons/network.py:retract`, `assert_node`, `challenge`, `add_justification` — public methods that invoke `_propagate`.

## 5. Flow

Every test follows the same 4-step pattern:

```
1. BUILD    — net = Network(); net.add_node(...); optionally add justifications
2. CORRUPT  — net.nodes["a"].dependents.add("ghost")   # or .update({...})
3. TRIGGER  — net.retract("a")  /  net.assert_node("a")  /  net.challenge(...)
4. ASSERT   — check net.nodes[...].truth_value, net.log, and/or the returned `changed` set
```

**Internal propagation flow (what's being tested):**
```
retract("a")
  → set a.truth_value = "OUT"
  → _propagate(start="a", direction=RETRACT)
      → for dep_id in a.dependents:          # {"nonexistent"} or {"b", "ghost"}
            if dep_id not in net.nodes:      # ← the guard under test
                log warning; continue        # skip, don't crash
            if dep_id in visited: continue
            visited.add(dep_id)
            → recursively propagate to dep
```

The test in `test_dangling_does_not_enter_visited` specifically confirms the guard's `continue` happens **before** `visited.add(dep_id)`, so a later-created real node with that ID won't be incorrectly skipped.

## 6. Invariants

1. **No KeyError on missing dependent**: `Network._propagate` must never raise `KeyError` due to a missing node in the `dependents` set. (Issue #22.)
2. **Warning is mandatory, not optional**: Every dangling reference must produce exactly one `net.log` entry with `action="warn"`, `target=<ghost_id>`, a `value` containing `"dangling"` and the parent ID, and a `timestamp`.
3. **Propagation completeness**: The presence of dangling refs must not reduce the set of nodes that receive state transitions. All valid dependents (and their transitive dependents) must still be processed.
4. **`changed` purity**: The return value of `retract` / `assert_node` must contain only node IDs that actually exist and changed state. Ghost IDs are excluded.
5. **`visited` purity**: Dangling IDs must not be added to the algorithm's `visited` set, preserving correctness if a node with that ID is later created.
6. **Path uniformity**: The guard must be active on *every* code path that reaches `_propagate` — `retract`, `assert_node`, `challenge`, `add_justification`.

## 7. Error Handling

This file tests the **absence** of an error (no `KeyError`) and the **presence** of a controlled, structured warning. Specifically:

- **Swallowed exception**: The `KeyError` that would occur from `net.nodes["nonexistent"]` is caught (presumably via `if dep_id not in self.nodes` or a `try/except`) inside `_propagate`. The tests verify the exception is *not* propagated to the caller.
- **Structured log as error channel**: Instead of raising, the system appends to `net.log`. The log entry is a plain dict (not an exception object), meaning downstream consumers parse the log rather than catching exceptions.
- **No silent failure**: The tests assert the warning *is* logged, preventing a regression where the guard simply `pass`es without observability.
- **No partial propagation**: The "continues" test class ensures the guard doesn't `break` the loop — it `continue`s, so subsequent valid dependents are still processed.

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class and its `_propagate` method; this is the production code that every assertion in this file is pinning down. Understanding the BFS/DFS traversal, the `visited` set, and the log-emission logic is prerequisite to modifying either the guard or the tests.
- [file] `reasons/network.py` (search for `_propagate`) — The private propagation method itself. Note how it iterates `node.dependents`, the exact `if dep_id not in self.nodes` check, and where the warning dict is constructed.
- [file] `tests/test_dependents_integrity.py` — A sibling test file that likely covers the *integrity* of the `dependents` set (e.g., ensuring it's kept in sync when nodes are added/removed). Together with this file, it defines the full contract for the dependents relationship.
- [repo] `ftl-reasons` issue tracker (#22) — The original bug report that motivated this guard. Understanding the production scenario that produced dangling references (e.g., concurrent deletion, import corruption, serialization round-trip) clarifies why the tests inject state directly rather than simulating a realistic corruption path.
- [general] `log-schema-contract` — The `net.log` entry structure (`action`, `target`, `value`, `timestamp`) is implicitly defined by these assertions. Any change to the log schema must update both `network.py` and every test that inspects `net.log`. Search for other test files that assert on `net.log` to map the full schema.

---

## Beliefs

- `dangling-dependent-guard-prevents-keyerror` — `Network._propagate` checks `if dep_id not in self.nodes` before indexing, so a missing node in `dependents` produces a warning log entry and a `continue`, never a `KeyError`.
- `dangling-ref-excluded-from-visited-set` — The guard's `continue` fires before `visited.add(dep_id)`, guaranteeing that a node ID that was dangling at propagation time can still be processed normally if it is created later.
- `warning-log-contains-parent-id` — Every dangling-dependent warning entry's `value` field includes the ID of the parent node whose `dependents` set contained the ghost, enabling operators to trace the corruption source.
- `propagation-path-uniformity-requires-guard-on-all-callers` — Because `retract`, `assert_node`, `challenge`, and `add_justification` all delegate to the same `_propagate`, a single guard implementation covers all public entry points; no caller is expected to pre-filter dangling refs.
- `changed-list-contains-only-existing-nodes` — The `changed` collection returned by `retract`/`assert_node` is a pure subset of `net.nodes.keys()`; dangling IDs never appear, making it safe to use as a direct work-order for downstream side-effects (pub/sub, persistence).
