# File: tests/test_dependents_integrity.py

**Date:** 2026-04-24
**Time:** 16:57

## Purpose

This file is a **structural integrity test suite** for the `dependents` reverse index on `Network` nodes. In the reasons TMS, each node tracks a forward relationship (justifications reference antecedents/outlist) and a reverse relationship (`node.dependents` — which other nodes depend on me). These two must stay in sync. This file owns the responsibility of proving that:

1. `_rebuild_dependents()` correctly reconstructs the reverse index from scratch.
2. `verify_dependents()` detects inconsistencies (extra or missing entries) without mutating state.
3. Every mutation path in `Network` (`add_node`, `retract`, `assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`) leaves the index consistent afterward.
4. Corruption can be detected and repaired.
5. The index survives SQLite round-trips via `Storage`.

## Key Components

### `assert_clean(net: Network)`
A one-line test helper that calls `net.verify_dependents()` and asserts the result is an empty list. Used as the universal consistency check throughout the file — every test that cares about index integrity calls this rather than reimplementing the check.

### `TestRebuildDependents`
Tests `_rebuild_dependents()` in isolation across graph shapes: empty, premises-only, single dependency, outlist-only, mixed antecedent+outlist, and stale/phantom entries. The key test is `test_rebuild_clears_stale_entries` — it injects a fake dependent `"phantom"`, confirms `verify_dependents` catches it, then confirms `_rebuild_dependents` removes it.

### `TestVerifyDependents`
Tests the read-only verifier. Validates it returns `[]` for clean graphs, reports `"extra"` for phantom entries, reports `"missing"` for dropped entries, and — critically — `test_does_not_modify_state` proves the verifier is side-effect-free (the ghost entry survives after verification).

### `TestMutationPathIntegrity`
The most important class. Exercises every `Network` mutation and asserts `assert_clean` afterward. This is the **contract**: any new mutation added to `Network` should get a corresponding test here. Covers: `add_node`, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`.

### `TestCorruptionAndRepair`
Deliberately corrupts the index (adds bogus entries, clears real entries), confirms `verify_dependents` catches it, then confirms `_rebuild_dependents` repairs it. Also proves rebuild is idempotent — calling it twice produces identical state.

### `TestStorageRoundTripDependents`
Saves a network to SQLite via `Storage`, loads it back, and asserts the dependents index is clean and has the correct entries. Tests both a simple graph and a diamond pattern (A→B, A→C, B+C→D).

### `TestEdgeCases`
Reviewer-flagged boundary conditions: dangling antecedent references (justification points to nonexistent node), duplicate antecedents across multiple justifications on the same node, self-referencing justifications, and the `_rewrite_dependents` API helper (used by deduplication) for both antecedent and outlist rewrites.

## Patterns

**Verify-then-repair pattern**: The suite separates detection (`verify_dependents`) from correction (`_rebuild_dependents`). Tests first prove detection works, then prove repair works, then prove mutations don't need repair. This mirrors the production flow — you can cheaply verify in a health check without the cost of a full rebuild.

**Mutation coverage matrix**: `TestMutationPathIntegrity` is structured as an exhaustive enumeration of Network's mutation surface. This is a common pattern for invariant testing — if someone adds a new mutation to `Network`, this class is where the corresponding test belongs.

**Corruption injection**: Tests deliberately break internal state (`dependents.add("ghost")`, `dependents.clear()`) to prove that the verification and repair machinery works. This is the TMS equivalent of a chaos test.

**Shared assertion helper**: `assert_clean` normalizes the "is the index valid?" check so test failures produce uniform, descriptive messages.

## Dependencies

**Imports from:**
- `reasons_lib.Justification` — the SL justification dataclass
- `reasons_lib.network.Network` — the dependency graph (the system under test)
- `reasons_lib.storage.Storage` — SQLite persistence (for round-trip tests)
- `reasons_lib.api._rewrite_dependents` — internal helper for deduplication rewrites

**Imported by:** Nothing — this is a leaf test module.

## Flow

Each test follows the same pattern:
1. Create a fresh `Network()` (no persistence, pure in-memory).
2. Build a graph by adding nodes with justifications.
3. Either (a) call `assert_clean` to verify the mutation left the index consistent, or (b) inject corruption, verify it's detected, repair it, verify it's clean.

The storage round-trip tests add a save→load cycle in the middle, using a `tempfile.TemporaryDirectory` for the SQLite database.

## Invariants

The central invariant this file enforces: **for every node N that appears in any justification's `antecedents` or `outlist` of node M, `N.dependents` must contain M — and nothing else.** Put differently, `dependents` is the exact reverse index of all justification references.

Secondary invariants:
- `verify_dependents()` is read-only — it never modifies the graph.
- `_rebuild_dependents()` is idempotent — running it twice produces the same result.
- `convert_to_premise` removes the node from its former antecedents' dependents sets (the old justification edges are gone, so the reverse index must reflect that).

## Error Handling

There is no error handling in the traditional sense. The file uses `assert` exclusively. `verify_dependents()` returns a list of human-readable error strings (containing keywords like `"extra"` and `"missing"`), which tests pattern-match against with `any("extra" in e ...)`. This design means verification failures are informational, not exceptional — the caller decides whether to repair or raise.

## Topics to Explore

- [function] `reasons_lib/network.py:verify_dependents` — The production implementation that this suite exercises; understanding what error strings it emits clarifies the `any("extra" in e ...)` assertions.
- [function] `reasons_lib/network.py:_rebuild_dependents` — The repair function; worth reading to see how it clears and reconstructs from justification references.
- [function] `reasons_lib/api.py:_rewrite_dependents` — Used by deduplication to redirect all justification references from one node ID to another while keeping the reverse index consistent.
- [file] `tests/test_dangling_dependents.py` — A related test file likely focused on the dangling-reference edge case; comparing the two clarifies the boundary of each file's responsibility.
- [function] `reasons_lib/network.py:convert_to_premise` — The mutation that removes justifications and must also clean up the reverse index; an interesting case because it's the only mutation that shrinks the dependents set.

## Beliefs

- `verify-dependents-is-readonly` — `verify_dependents()` never modifies `node.dependents`; it only reads and reports discrepancies as a list of strings.
- `rebuild-dependents-is-idempotent` — Calling `_rebuild_dependents()` twice in succession produces identical `dependents` sets on every node.
- `every-network-mutation-maintains-dependents` — After any public mutation method on `Network` (`add_node`, `retract`, `assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`), `verify_dependents()` returns an empty list.
- `dependents-survive-storage-roundtrip` — After `Storage.save()` followed by `Storage.load()`, the loaded network's dependents index passes `verify_dependents()` with no errors.
- `convert-to-premise-removes-dependents` — When a derived node is converted to a premise via `convert_to_premise`, it is removed from its former antecedents' `dependents` sets.

