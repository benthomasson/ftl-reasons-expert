---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_dependents_integrity.py

# `tests/test_dependents_integrity.py`

## 1. Purpose

This file is the **invariant enforcement test suite** for the dependents reverse index in the `ftl-reasons` belief network. In a defeasible-logic (FtL) system, every node stores a `dependents` set — the collection of node IDs that *depend on* it (via antecedents or outlist references in their justifications). This reverse index is the backbone of propagation: when a belief is retracted, superseded, or challenged, the network walks `dependents` to find everything that must be re-evaluated.

This test file owns the responsibility of proving that:

- The index is **correctly built** (`_rebuild_dependents`)
- The index is **correctly verified** (`verify_dependents`)
- Every **mutation path** in `Network` leaves the index consistent
- **Corruption** (phantom entries, missing entries) is detectable and repairable
- **Persistence** (save/load round-trip) preserves the index
- **Edge cases** (dangling references, self-loops, duplicate justifications) don't break the invariant

## 2. Key Components

### Helper: `assert_clean(net: Network)`
```python
def assert_clean(net: Network):
    errors = net.verify_dependents()
    assert errors == [], f"Dependents index inconsistent: {errors}"
```
A one-liner assertion wrapper. It calls `net.verify_dependents()` and fails the test with a descriptive message listing all inconsistencies. Used ~15 times across the file.

### `TestRebuildDependents`
Tests `_rebuild_dependents()` in isolation. Key contracts it verifies:

| Test | Contract |
|------|----------|
| `test_empty_network` | Rebuild on zero nodes is a no-op that passes verification |
| `test_premises_only` | Nodes with no justifications get `dependents == set()` |
| `test_single_dependency` | Antecedent `a` gets `"b"` in its dependents set |
| `test_outlist_dependency` | Outlist entries also create reverse links (defeater semantics) |
| `test_both_antecedent_and_outlist` | A single justification can create links from *both* antecedents and outlist |
| `test_rebuild_clears_stale_entries` | Rebuild is **destructive-then-rebuild**: it clears all existing dependents before populating |

### `TestVerifyDependents`
Tests `verify_dependents()` as a **read-only auditor**:

- Returns `[]` on a clean network
- Returns error strings containing `"extra"` + the phantom ID for spurious entries
- Returns error strings containing `"missing"` + the absent ID for missing entries
- **Does not mutate state** — `test_does_not_modify_state` injects a `"ghost"` entry, calls verify, and confirms the ghost is still there

### `TestMutationPathIntegrity`
The broadest class. Each test exercises one `Network` mutation and then calls `assert_clean`:

- `add_node` with justifications
- `retract` → `assert_node` cycle
- `add_justification` (post-hoc justification addition)
- `supersede` (old belief is replaced by new)
- `challenge` / `defend` (dialectical moves)
- `convert_to_premise` (removes justifications, should remove reverse links)
- `add_nogood` (inconsistency detection)
- `summarize` (meta-node depending on multiple sources)

### `TestCorruptionAndRepair`
The "detect → repair" pattern:
- `test_detect_and_repair_extra`: Injects a bogus reverse entry → `verify` catches it → `_rebuild_dependents` fixes it
- `test_detect_and_repair_missing`: Clears all dependents on `a` → `verify` catches it → rebuild fixes it
- `test_rebuild_idempotent`: Calling `_rebuild_dependents` twice produces identical state (no drift)

### `TestStorageRoundTripDependents`
Uses `Storage` (SQLite-backed) with a `tempfile.TemporaryDirectory`:
- `test_round_trip_preserves_dependents`: Mixed antecedent + outlist graph survives save/load
- `test_round_trip_complex_graph`: Diamond topology (A→B, A→C, B+C→D) with exact set assertions on every node's dependents

### `TestEdgeCases`
Reviewer-flagged boundary conditions:
- **Dangling antecedent**: A justification references `"missing"` (a node that was never added). The network must not crash and must still pass verification.
- **Duplicate antecedent**: Two justifications on the same node both list `"a"`. The dependents set should contain `"b"` exactly once (set semantics).
- **Self-reference**: Node `"a"` lists itself as its own antecedent. The test asserts this is *allowed* and passes verification (degenerate but valid).
- **`_rewrite_dependents` (antecedents)**: The API-level deduplication helper rewrites `"old"` → `"new"` in justifications *and* in the dependents index.
- **`_rewrite_dependents` (outlist)**: Same, but for outlist references.

## 3. Patterns

- **Arrange-Act-Assert**: Every test follows this. The "arrange" is building a small `Network` graph; "act" is the mutation under test; "assert" is `assert_clean` plus specific set-membership checks.
- **Destructive-then-rebuild invariant**: `_rebuild_dependents` is tested to clear *all* existing dependents first (`test_rebuild_clears_stale_entries` injects a `"phantom"` and confirms it's gone after rebuild). This means the method is safe to call at any time as a repair mechanism.
- **Read-only verification**: `verify_dependents` is explicitly tested to be side-effect-free (`test_does_not_modify_state`). This separation (audit vs. repair) is a deliberate architectural choice.
- **Table-driven edge cases**: `TestEdgeCases` groups boundary conditions that were flagged during code review, suggesting these tests were added reactively to close gaps.
- **Temporary directory isolation**: Storage tests use `tempfile.TemporaryDirectory()` as a context manager, ensuring no residual DB files leak between tests.
- **No shared fixtures**: Each test constructs its own `Network` from scratch. There's no class-level or module-level fixture creating a shared graph. This keeps tests independent and order-insensitive.

## 4. Dependencies

### Imports (this file depends on)
| Import | Role |
|--------|------|
| `reasons.Justification` | Dataclass for a single inference rule (type, antecedents, outlist) |
| `reasons.network.Network` | The core belief graph; provides `add_node`, `retract`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`, `add_justification`, `_rebuild_dependents`, `verify_dependents` |
| `reasons.storage.Storage` | SQLite persistence layer (`save`, `load`, `close`) |
| `reasons.api._rewrite_dependents` | Internal helper for dedup/rename that rewrites both justifications and the dependents index |
| `pytest` | Test runner (imported but no decorators visible — likely used for assertion rewriting and the runner itself) |
| `tempfile`, `pathlib.Path` | Isolated temp DB files for storage tests |

### What depends on this file
Nothing imports this test file. It's a leaf in the dependency graph. However, the **invariants it enforces** are depended upon by:
- `reasons/network.py` — the propagation logic that walks `dependents`
- `reasons/api.py` — `_rewrite_dependents` assumes the index is clean before and after
- `reasons/repair.py`, `reasons/repair_premises.py` — repair workflows that call `_rebuild_dependents`
- `reasons/storage.py` — the `load()` path must reconstruct a valid dependents index

## 5. Flow

```
Test execution (pytest collects classes in file order):

TestRebuildDependents
  └─ Build small graph → call net._rebuild_dependents() → assert_clean()
     Internally: _rebuild_dependents iterates all nodes, clears all .dependents sets,
     then for each node, for each justification, adds the node's ID to every
     antecedent's and outlist entry's .dependents set.

TestVerifyDependents
  └─ Build graph (or corrupt it) → call net.verify_dependents() → assert error strings
     Internally: verify_dependents re-derives the expected dependents from justifications
     and diffs against the stored .dependents sets. Returns list of error strings.

TestMutationPathIntegrity
  └─ Build graph → call one mutation (retract, supersede, challenge, etc.) → assert_clean()
     Each mutation is responsible for maintaining the index incrementally.
     The test verifies no incremental path leaves the index dirty.

TestCorruptionAndRepair
  └─ Build clean graph → manually corrupt (add/remove entries) → verify catches it
     → _rebuild_dependents repairs → assert_clean()
     Demonstrates the "detect-then-repair" operational pattern.

TestStorageRoundTripDependents
  └─ Build graph → Storage.save(net) → Storage.load() → assert_clean() on loaded graph
     Proves the serialization format preserves (or correctly reconstructs) dependents.

TestEdgeCases
  └─ Degenerate inputs (dangling refs, self-loops, duplicates) → assert no crash + clean index
     Also tests _rewrite_dependents from reasons.api for rename/dedup scenarios.
```

The data flow in each test is: **construct → mutate → verify/repair → assert**. There's no inter-test data flow.

## 6. Invariants

The file enforces (and documents) these invariants:

1. **Reverse index completeness**: For every justification `J` on node `X` with antecedent `A`, the set `nodes[A].dependents` must contain `X`. Same for outlist entries.
2. **Reverse index soundness**: `nodes[A].dependents` must not contain any node `X` where no justification on `X` references `A`. (No phantom entries.)
3. **Rebuild atomicity**: `_rebuild_dependents` is all-or-nothing — it clears *all* dependents before rebuilding. A partial failure can't leave a mixed state.
4. **Verify immutability**: `verify_dependents` is a pure observer. It never mutates `nodes[*].dependents`.
5. **Rebuild idempotency**: Calling `_rebuild_dependents` on an already-correct network is a no-op (same output).
6. **Set semantics**: The dependents field is a `set`. Duplicate justifications referencing the same antecedent must not create duplicate entries (enforced by `test_multiple_justifications_same_antecedent`).
7. **Storage fidelity**: After `save` → `load`, the dependents index must be byte-for-byte equivalent (as sets) to the original.
8. **Dangling tolerance**: A justification referencing a non-existent node ID must not crash `_rebuild_dependents` or `verify_dependents`. The index is only built from nodes that *exist* in `net.nodes`.

## 7. Error Handling

This is a test file, so "error handling" means **how errors are asserted**:

- **`verify_dependents()` returns a list of strings**, not exceptions. The tests check `len(errors) > 0` for corruption detection and `errors == []` for cleanliness. Error strings are expected to contain human-readable keywords (`"extra"`, `"missing"`) and the offending node ID.
- **`assert_clean`** wraps the verification in an `assert` with a formatted message: `f"Dependents index inconsistent: {errors}"`. This gives the developer the full list of inconsistencies on failure.
- **No `try/except` blocks** in any test. The tests rely on either assertion failures (for expected errors) or pytest's natural failure on unhandled exceptions (for unexpected crashes).
- **Storage errors** are not explicitly tested here — if `Storage.save` or `Storage.load` raises, the test fails with that exception. The file assumes the storage layer handles its own error cases (tested in `test_storage.py` and `test_pg.py`).
- **`_rewrite_dependents`** is imported from `reasons.api` — if it raises on invalid input, the test fails. There's no explicit error-path test for that function in this file.

---

## Topics to Explore

- [function] `reasons/network.py:_rebuild_dependents` — The implementation this entire file validates; understanding its iteration order and clearing strategy is critical for maintaining the invariant
- [function] `reasons/network.py:verify_dependents` — The read-only auditor; its error-message format (`"extra"`, `"missing"`) is contractually depended upon by these tests
- [file] `reasons/api.py` — Contains `_rewrite_dependents`, the dedup/rename helper tested in `TestEdgeCases`; understanding how it coordinates justification rewriting with index updates reveals the broader mutation protocol
- [file] `tests/test_dangling_dependents.py` — A sibling test file that likely covers the same index from the "dangling reference" angle; comparing the two reveals whether edge-case coverage is split or overlapping
- [general] `dependents-propagation-semantics` — How `retract`, `supersede`, and `challenge` traverse the dependents index to trigger cascading re-evaluation; the invariant tested here is the *precondition* for that propagation to be correct

## Beliefs

- `verify-dependents-is-pure` — `verify_dependents()` returns a list of error strings and never mutates any node's `dependents` set; it is safe to call at any point as a non-destructive audit
- `rebuild-dependents-clears-before-populating` — `_rebuild_dependents()` first clears all `dependents` sets to `set()` before re-deriving them from justifications, making it safe as a repair mechanism regardless of prior corruption
- `outlist-entries-participate-in-dependents` — Nodes listed in a justification's `outlist` field are treated identically to `antecedents` entries for the purpose of the reverse dependents index; both create reverse links
- `dangling-antecedent-is-tolerated` — A justification referencing a node ID absent from `net.nodes` does not cause `_rebuild_dependents` or `verify_dependents` to raise; the nonexistent node simply contributes no reverse link
- `storage-roundtrip-preserves-dependents-as-sets` — After `Storage.save()` → `Storage.load()`, every node's `dependents` is set-equal to the original, meaning the serialization format either stores the index explicitly or reconstructs it deterministically from justifications
