# File: tests/test_dangling_dependents.py

**Date:** 2026-04-29
**Time:** 16:53

# `tests/test_dangling_dependents.py`

## Purpose

This file is the regression test suite for **issue #22** — a bug where the `_propagate` method in `Network` would crash with a `KeyError` when a node's `dependents` set contained IDs that didn't correspond to any node in `net.nodes`. The fix was to add a guard that skips dangling references and emits a warning log entry instead of crashing. These tests lock that behavior down.

The file owns verification of a single invariant: **propagation is resilient to stale/orphaned entries in `dependents` sets**.

## Key Components

Five test classes, each covering a distinct aspect of the dangling-dependent guard:

### `TestDanglingDependentNoCrash`
The most basic contract: calling `retract`, `assert_node`, or any propagation path with dangling refs in `dependents` **must not raise**. Four tests exercise single danglers, multiple danglers, and the case where valid and dangling dependents coexist. The valid-dependent test also asserts that real propagation still works (node `b` goes OUT when `a` is retracted).

### `TestDanglingDependentWarningLogs`
Verifies the **observability contract**: every dangling reference produces exactly one warning entry in `net.log` with `action == "warn"`, a `target` field naming the ghost node, and a `value` string containing `"dangling"` plus the parent node ID. Also checks that warnings carry a `timestamp`. This class tests both the `retract` and `assert_node` code paths.

### `TestDanglingDependentPropagationContinues`
The **correctness-under-failure** contract: after encountering a dangling ref, propagation must continue to all reachable valid nodes. Tests cover linear chains (A→B), chains with a dangling ref in the middle (A→B→C where B has a ghost), and a diamond topology (A→B, A→C, B+C→D). Each verifies that the `changed` set includes all valid nodes and excludes ghosts.

### `TestDanglingDependentEdgeCases`
Corner cases that could regress:
- **Same ghost ID in two nodes** — warns at least once per encounter.
- **Ghost not in `changed`** — dangling refs must never leak into the return value.
- **`challenge()` path** — `challenge` calls `_propagate` internally; the guard must fire there too.
- **`add_justification()` path** — another indirect trigger of `_propagate`.
- **Ghost does not enter `visited`** — if the ghost ID is later added as a real node, it must not be incorrectly skipped by a stale visited-set entry. This is the subtlest test: it validates that the guard's skip logic doesn't poison future propagation state.

## Patterns

**Synthetic corruption**: Every test manually injects dangling refs via `net.nodes["a"].dependents.add("ghost")` rather than reaching the state through normal API calls. This is deliberate — the bug occurs when the `dependents` index gets out of sync with `net.nodes`, which shouldn't happen through the public API but can happen through storage round-trips or concurrent modifications.

**Log-as-assertion-target**: The tests query `net.log` (a list of dicts) to verify warnings were emitted, treating the structured log as a first-class test output. This avoids coupling to Python's `logging` module or `capsys`.

**Topology progression**: Tests escalate from trivial (single node + ghost) to complex (diamond graph + ghost), a common pattern in graph-algorithm test suites.

## Dependencies

**Imports:**
- `Justification`, `Node` from `reasons_lib` — `Node` is imported but unused (likely a leftover); `Justification` is used to build SL justification records for derived nodes.
- `Network` from `reasons_lib.network` — the core in-memory TMS graph; all tests instantiate it directly.

**Imported by:** Nothing — this is a leaf test module.

**Implicit dependency:** The tests depend on `Network` exposing `.nodes` (dict), `.log` (list of dicts), `.retract()`, `.assert_node()`, `.challenge()`, `.add_justification()`, and `.add_node()`. Changes to any of these signatures will break tests here.

## Flow

Each test follows the same pattern:

1. **Build** a small network with `add_node` calls and SL justifications.
2. **Corrupt** a node's `dependents` set by inserting ghost IDs.
3. **Trigger propagation** via `retract`, `assert_node`, `challenge`, or `add_justification`.
4. **Assert** one or more of: no exception raised, correct truth values on valid nodes, correct `changed` set contents, correct warning log entries.

There is no shared fixture — each test creates its own `Network()`, keeping tests fully isolated.

## Invariants

1. **No KeyError on dangling ref**: `_propagate` must skip any dependent ID not present in `net.nodes`.
2. **One warning per dangling encounter**: Each ghost reference produces exactly one log entry with `action="warn"`.
3. **Ghost excluded from `changed`**: The return value of `retract`/`assert_node` never contains dangling IDs.
4. **Ghost excluded from `visited`**: The propagation visited set must not record dangling IDs, so they don't interfere with later real-node propagation.
5. **Valid propagation unaffected**: The presence of dangling refs must not prevent any reachable valid node from being updated.

## Error Handling

The tests validate that errors are **not** produced — the whole point is that `_propagate` swallows the missing-node case gracefully. The guard converts what would be a `KeyError` crash into a structured warning log entry. The tests verify the warning format but don't test any exception-raising paths, because none should exist for this scenario.

## Topics to Explore

- [function] `reasons_lib/network.py:_propagate` — The actual guard implementation: how it detects dangling refs, emits warnings, and continues traversal
- [file] `tests/test_dependents_integrity.py` — Likely tests the other side of this problem: ensuring `dependents` stays consistent under normal operations
- [function] `reasons_lib/network.py:retract` — Entry point that triggers `_propagate`; understanding the retraction algorithm explains why dangling refs are encountered
- [general] `outlist-dependents-tracking` — The CLAUDE.md notes that outlist nodes aren't tracked in the dependents index, which is a related consistency gap that could produce similar dangling-ref scenarios
- [file] `reasons_lib/network.py` — The `Node` class definition and `dependents` set management; understanding how dependents are added/removed during normal `add_node`/`add_justification` explains how they can get out of sync

## Beliefs

- `dangling-dependent-guard-skips-missing-nodes` — `_propagate` skips dependent IDs not present in `net.nodes` rather than raising `KeyError`, and emits a structured warning log entry for each
- `dangling-refs-excluded-from-changed-set` — The `changed` set returned by `retract`/`assert_node` never contains node IDs that don't exist in the network
- `dangling-refs-excluded-from-visited-set` — Dangling dependent IDs are not added to the propagation visited set, so later-created nodes with the same ID propagate normally
- `unused-node-import-in-dangling-test` — `Node` is imported from `reasons_lib` but never used in the test file
- `warning-log-contract-action-target-value` — Dangling-dependent warnings in `net.log` have `action="warn"`, a `target` field with the ghost ID, and a `value` string containing both `"dangling"` and the parent node ID

