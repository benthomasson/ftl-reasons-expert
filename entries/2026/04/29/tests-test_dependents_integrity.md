# File: tests/test_dependents_integrity.py

**Date:** 2026-04-29
**Time:** 16:57

## Purpose

`test_dependents_integrity.py` is a comprehensive property-based test suite that guards a single invariant: **the `dependents` reverse index on every `Network` node must exactly mirror the forward references in all justifications**. If node B's justification lists node A as an antecedent or outlist member, then A's `dependents` set must contain B — no more, no less.

This matters because the TMS's retraction cascade and truth-value propagation walk `dependents` to find affected nodes. A missing entry means a retraction silently skips a downstream belief; an extra entry means spurious re-evaluation or errors.

## Key Components

### `assert_clean(net)`
A one-line helper that calls `net.verify_dependents()` and fails if any inconsistencies are returned. Used as the central assertion throughout the file — every test either calls this directly or checks its preconditions.

### `TestRebuildDependents`
Tests `Network._rebuild_dependents()` — the canonical repair function. Covers:
- Empty network, premise-only, single dependency, outlist-only, mixed antecedent+outlist
- **Stale entry clearing**: injects a phantom dependent, confirms `_rebuild_dependents()` removes it

### `TestVerifyDependents`
Tests `Network.verify_dependents()` as a **read-only diagnostic**. Verifies it:
- Returns `[]` on a clean network
- Detects extra dependents (entries that shouldn't exist)
- Detects missing dependents (entries that should exist but don't)
- Does **not** modify state (the "ghost" entry survives the call)

### `TestMutationPathIntegrity`
The most important class. Every public mutation on `Network` gets a test confirming the dependents index stays clean *after* the mutation. Covered operations:
- `add_node` with justifications
- `retract` / `assert_node` (retract-then-restore cycle)
- `add_justification` (adding a new justification to an existing node)
- `supersede` (one belief replacing another)
- `challenge` / `defend` (dialectical operations)
- `convert_to_premise` (strips justifications, must remove old dependent links)
- `add_nogood` (recording contradictions)
- `summarize` (summary node depending on source nodes)

### `TestCorruptionAndRepair`
Deliberately corrupts the index (adds bogus entries, clears real ones), then verifies `verify_dependents()` catches it and `_rebuild_dependents()` fixes it. Also checks rebuild idempotency.

### `TestStorageRoundTripDependents`
Saves a network to SQLite via `Storage`, loads it back, and confirms the loaded network's dependents index is clean. Tests both a simple graph and a diamond dependency pattern (A → B, A → C, B+C → D).

### `TestEdgeCases`
Boundary conditions flagged during code review:
- **Dangling antecedent**: justification references a node not in the network
- **Duplicate antecedent**: two justifications on the same node both reference the same antecedent (dependents should still be a set, not double-counted)
- **Self-reference**: a node listing itself as its own antecedent
- **`_rewrite_dependents`**: the deduplication helper in `api.py` that rewrites all references from an old node ID to a new one, covering both antecedents and outlists

## Patterns

**Mutation-path coverage**: rather than testing `_rebuild_dependents` in isolation, the suite tests the *invariant after every mutation*. This is closer to a contract test than a unit test — it doesn't care how the code maintains the index, only that it's correct after each operation.

**Corruption injection**: tests deliberately break internal state (`dependents.add("phantom")`, `dependents.clear()`) to validate the diagnostic and repair paths. This is a classic pattern for testing self-healing data structures.

**Shared assertion helper**: `assert_clean()` provides a single point of failure reporting. If the error format from `verify_dependents()` changes, only this helper needs updating.

**No fixtures**: every test constructs its own `Network()` from scratch. This avoids shared mutable state between tests at the cost of some repetition — a deliberate tradeoff for test isolation.

## Dependencies

**Imports**:
- `reasons_lib.network.Network` — the core TMS graph being tested
- `reasons_lib.Justification` — the SL/CP justification dataclass
- `reasons_lib.storage.Storage` — SQLite persistence (round-trip tests only)
- `reasons_lib.api._rewrite_dependents` — internal helper for deduplication (edge-case tests only)
- `tempfile`, `pathlib.Path` — for ephemeral SQLite databases
- `pytest` — imported but not explicitly used (likely for future parametrization or markers)

**Nothing imports this file** — it's a test module, consumed only by pytest.

## Flow

Each test follows the same pattern:
1. Create a fresh `Network()`
2. Build a small graph via `add_node` calls with justifications
3. Optionally mutate the network (retract, supersede, challenge, etc.) or corrupt internal state
4. Assert the dependents index is clean via `assert_clean(net)`, or assert specific error messages from `verify_dependents()`

The storage round-trip tests add a save/load cycle between steps 2 and 4.

## Invariants

The file enforces one core invariant with multiple facets:

1. **Completeness**: if node X appears in any justification of node Y (antecedent or outlist), then `Y ∈ X.dependents`
2. **Minimality**: if `Y ∈ X.dependents`, there must exist a justification on Y that references X
3. **Mutation stability**: the invariant holds after every public `Network` method
4. **Persistence stability**: the invariant holds after a save/load round-trip
5. **Repair correctness**: `_rebuild_dependents()` restores the invariant from any corrupted state
6. **Diagnostic purity**: `verify_dependents()` is read-only — it reports errors without modifying state

## Error Handling

The tests don't test error handling per se — they test an integrity-checking mechanism. `verify_dependents()` returns a list of human-readable error strings (containing words like "extra" or "missing"), and the tests assert on those strings using substring matching (`any("extra" in e ... for e in errors)`). This is slightly fragile to message rewording but keeps the tests readable.

No exceptions are expected from any of the operations; if `add_node`, `retract`, or `_rebuild_dependents` raised, pytest would catch it as an unexpected failure.

## Topics to Explore

- [function] `reasons_lib/network.py:verify_dependents` — The diagnostic function these tests exercise; understanding its walk logic clarifies what "extra" vs "missing" means
- [function] `reasons_lib/network.py:_rebuild_dependents` — The canonical repair path; its implementation determines whether dependents are maintained incrementally or recomputed from scratch
- [function] `reasons_lib/api.py:_rewrite_dependents` — The deduplication rewriter tested in the edge-case section; a mutation path that lives outside `Network` itself
- [file] `tests/test_dangling_dependents.py` — A sibling test file likely focused on the dangling-reference scenario, which overlaps with some edge cases here
- [general] `outlist-dependent-tracking` — The CLAUDE.md documents a known issue where outlist nodes aren't tracked in the dependents index; these tests assert the opposite, suggesting a fix has landed

## Beliefs

- `verify-dependents-is-read-only` — `Network.verify_dependents()` never modifies the dependents index; it returns error strings without side effects
- `rebuild-dependents-clears-before-rebuilding` — `_rebuild_dependents()` wipes all existing dependent sets before recomputing from justifications, so stale entries are always removed
- `every-network-mutation-maintains-dependents-invariant` — After any public mutation (add_node, retract, assert_node, add_justification, supersede, challenge, defend, convert_to_premise, add_nogood, summarize), the dependents index passes `verify_dependents()`
- `storage-round-trip-preserves-dependents` — Saving a Network to SQLite via Storage and loading it back produces a clean dependents index matching the original
- `rewrite-dependents-updates-both-antecedents-and-outlists` — `_rewrite_dependents(net, old, new)` rewrites justification references and dependent sets for both antecedent and outlist occurrences of the old node ID

