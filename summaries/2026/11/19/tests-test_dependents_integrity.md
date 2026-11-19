---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_dependents_integrity.py

# Explanation: `tests/test_dependents_integrity.py`

## Purpose

This is a pytest test module for the **dependents reverse index** maintained by `reasons.network.Network`. It is a white-box integrity suite: it verifies that each node’s `dependents` set stays consistent as the graph is mutated, that corruption is detected, and that `_rebuild_dependents()` can repair it.

The file owns coverage for:

- `Network._rebuild_dependents()` — canonical rebuild/repair.
- `Network.verify_dependents()` — read-only consistency checking.
- Every known mutation path in `Network` that can affect antecedents/outlist references.
- Storage save/load round-trips.
- Reviewer-flagged edge cases such as dangling references, duplicate justifications, self-references, and `_rewrite_dependents()`.

It is not production code. Pytest discovers it via its `Test*` classes and `test_*` methods.

---

## Key Components

### `assert_clean(net: Network)`

A helper that centralizes the core invariant:

```python
errors = net.verify_dependents()
assert errors == [], f"Dependents index inconsistent: {errors}"
```

Contract: `verify_dependents()` must return an empty list for a healthy network.

---

### `TestRebuildDependents`

Tests `_rebuild_dependents()` as the canonical rebuild operation.

Covered cases:

- Empty network.
- Premises only: no dependents.
- Single antecedent dependency.
- `outlist` dependency.
- Both `antecedents` and `outlist` on one justification.
- Stale entries are cleared before rebuilding.

Key contract: `_rebuild_dependents()` reconstructs `dependents` purely from node justifications, clearing any previous state.

---

### `TestVerifyDependents`

Tests `verify_dependents()` as a detector, not a fixer.

Covered cases:

- Clean network returns `[]`.
- Extra dependent entry is detected.
- Missing dependent entry is detected.
- `verify_dependents()` does not mutate state.

The tests rely on specific substrings in error messages:

```python
assert any("extra" in e and "ghost" in e for e in errors)
assert any("missing" in e and "b" in e for e in errors)
```

So maintainers should preserve `"extra"` and `"missing"` wording, or update these tests.

---

### `TestMutationPathIntegrity`

Verifies that normal graph mutations leave the dependents index clean.

Covered mutation paths:

- `add_node(..., justifications=[...])`
- `retract()` and `assert_node()`
- `add_justification()`
- `supersede()`
- `challenge()`
- `challenge()` followed by `defend()`
- `convert_to_premise()`
- `add_nogood()`
- `summarize(..., over=[...])`

Examples of asserted reverse edges:

- After `add_justification("c", Justification(..., antecedents=["b"]))`, `"c"` must appear in `net.nodes["b"].dependents`.
- After `summarize("summary", ..., over=["a", "b"])`, `"summary"` must appear in both `"a"` and `"b"` dependents.
- After `convert_to_premise("b")`, `"b"` must no longer appear in `"a"` dependents.

---

### `TestCorruptionAndRepair`

Simulates corruption by directly mutating `Node.dependents`, then verifies repair.

Cases:

- Extra bogus reverse entry → detected → rebuilt clean.
- Missing reverse entry → detected → rebuilt clean.
- `_rebuild_dependents()` is idempotent: two rebuilds produce identical dependents snapshots.

This confirms that `dependents` is treated as **derived data** whose source of truth is each node’s justifications.

---

### `TestStorageRoundTripDependents`

Tests `Storage.save(net)` and `Storage.load()` using a temporary SQLite-like DB path:

```python
with tempfile.TemporaryDirectory() as td:
    db_path = Path(td) / "test.db"
    store = Storage(db_path)
    store.save(net)
    loaded = store.load()
    store.close()
```

Covered cases:

- Simple graph with antecedent and outlist references survives save/load with clean dependents.
- Diamond graph `A -> B`, `A -> C`, `B+C -> D` reloads with exact expected `dependents` sets.

---

### `TestEdgeCases`

Reviewer-oriented cases:

- Dangling antecedent reference to a missing node is tolerated and does not corrupt the index.
- Multiple justifications referencing the same antecedent deduplicate through set semantics.
- Self-referencing justification is accepted when the corresponding self-dependent entry exists.
- `reasons.api._rewrite_dependents()` rewrites antecedent references and moves reverse-index entries.
- `_rewrite_dependents()` also rewrites `outlist` references.

The `_rewrite_dependents` tests import the private helper locally:

```python
from reasons.api import _rewrite_dependents
```

---

## Patterns

- **Class-based pytest grouping**: behavior is grouped into `Test*` classes rather than standalone functions.
- **Arrange–Act–Assert** in nearly every test.
- **Centralized invariant helper**: `assert_clean()` avoids repeating the `verify_dependents()` assertion.
- **White-box corruption testing**: tests directly mutate `net.nodes[id].dependents` to simulate index drift.
- **Derived-data testing**: source of truth is `Node.justifications`; `dependents` is the reverse index.
- **Round-trip integration testing**: storage tests save and reload a `Network` through a real `Storage` object and temp database.
- **Private API coverage**: the suite intentionally exercises `_rebuild_dependents`, `_rewrite_dependents`, and internal node fields.

---

## Dependencies

### Imports used by the file

- `tempfile` — temporary directory for storage tests.
- `pathlib.Path` — database path construction.
- `pytest` — imported but not visibly used in this file.
- `reasons.Justification` — constructs justifications with `antecedents` and `outlist`.
- `reasons.network.Network` — primary object under test.
- `reasons.storage.Storage` — save/load round-trip tests.
- `reasons.api._rewrite_dependents` — imported inside edge-case tests.

The tests import from `reasons`, not `reasons_lib`, so they exercise the top-level `reasons` package present in this repository.

### What depends on this file

Only the test runner depends on it. Production modules do not import this test file.

### Internal coupling

The tests are tightly coupled to implementation details:

- `net.nodes[id].dependents`
- `net.nodes[id].justifications`
- `Network._rebuild_dependents()`
- `Network.verify_dependents()`
- `reasons.api._rewrite_dependents()`

Changes to those names, return types, or error-message wording will require test updates.

---

## Flow

Most tests follow this pattern:

1. Create a fresh `Network()`.
2. Add nodes and justifications in a controlled graph shape.
3. Perform a mutation or directly corrupt `dependents`.
4. Call `assert_clean()`, `verify_dependents()`, or `_rebuild_dependents()`.
5. Assert specific reverse-index memberships.

Storage tests add an external lifecycle:

1. Build a graph in memory.
2. Create a temporary directory and database path.
3. `Storage.save(net)`.
4. `Storage.load()` into a new network.
5. `store.close()`.
6. Assert the loaded network is clean and has expected `dependents`.

`_rewrite_dependents` tests:

1. Build a graph where one node references `"old"`.
2. Call `_rewrite_dependents(net, "old", "new")`.
3. Assert the justification now references `"new"`.
4. Assert the reverse-index entry moved from `"old"` to `"new"`.
5. Assert the whole index is clean.

---

## Invariants

This file enforces or documents these invariants:

1. **Dependents are derived from justifications.**
   For an existing node `x`, `x.dependents` should contain every node `n` whose justification references `x` through either `antecedents` or `outlist`.

2. **`_rebuild_dependents()` is canonical and idempotent.**
   It clears existing dependents and reconstructs them from justifications. Running it twice yields the same result.

3. **`verify_dependents()` is read-only.**
   It returns a list of consistency errors and must not mutate `dependents`.

4. **Error categories are distinguishable by message.**
   Extra reverse entries contain `"extra"`; missing reverse entries contain `"missing"`.

5. **All normal mutation paths preserve consistency.**
   `add_node`, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, and `summarize` are all expected to leave the index clean.

6. **Outlist references count as dependencies.**
   A node referencing another via `outlist=["a"]` creates a reverse edge from `"a"` to the referencing node.

7. **Dangling references are tolerated.**
   A justification may reference a missing node ID. Since the missing node is absent from `net.nodes`, it is ignored by the index rather than creating a phantom entry.

8. **Duplicate references collapse through set semantics.**
   Multiple justifications referencing the same antecedent produce one dependent entry.

9. **Self-references are acceptable when indexed consistently.**
   The edge-case test manually adds both a self-referencing justification and the matching self-dependent entry and expects the index to be considered clean.

10. **Storage save/load preserves dependents.**
    A loaded network should have a clean, equivalent dependents index.

---

## Error Handling

- `verify_dependents()` is designed to **report**, not raise. It returns a list of inconsistency strings.
- `assert_clean()` converts a non-empty error list into an `AssertionError` with a useful message.
- Tests intentionally corrupt state by directly mutating `dependents`; the corruption itself does not raise. Detection happens through `verify_dependents()`.
- Storage exceptions are not explicitly caught. If `Storage.save()`, `Storage.load()`, or `Storage.close()` raises, pytest fails naturally.
- `TemporaryDirectory` cleans up the database directory via context manager, but the `Storage` object is not used as a context manager in these tests.

---

## Topics to Explore

- [file] `reasons/network.py` — Contains `Network`, node mutation methods, and the dependents-index implementation exercised by this suite.
- [function] `reasons/network.py:_rebuild_dependents` — Inspect the canonical rebuild scanner, especially handling of antecedents, outlist, dangling IDs, and self-references.
- [function] `reasons/network.py:verify_dependents` — Review exact error-string formatting and read-only comparison logic.
- [function] `reasons/api.py:_rewrite_dependents` — The deduplication rewrite helper directly covered by the edge-case tests.
- [file] `tests/test_storage.py` — Broader storage behavior to complement the save/load dependents round-trip coverage.

---

## Beliefs

- `dependents-index-is-derived-from-justifications` — A node’s `dependents` set must match the nodes whose justifications reference it via `antecedents` or `outlist`.
- `rebuild-dependents-is-canonical-repair` — `_rebuild_dependents()` clears and reconstructs all dependents from justifications and is idempotent.
- `verify-dependents-is-read-only` — `verify_dependents()` returns a list of `"extra"`/`"missing"` errors without modifying the index.
- `mutation-methods-preserve-dependents-integrity` — `add_node`, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`/`defend`, `convert_to_premise`, `add_nogood`, and `summarize` are covered and leave the index clean.
- `rewrite-dependents-updates-both-edges` — `_rewrite_dependents()` rewrites justification antecedents/outlist references and moves reverse-index entries from the old node ID to the new one.
