---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_storage.py

# `tests/test_storage.py` — SQLite Persistence Tests

## 1. Purpose

This file validates that the `Storage` class (the SQLite persistence layer for the defeasible reasoning network) correctly **round-trips** a `Network`'s full state. Its single responsibility is: *"If you save a network to disk and load it back, the resulting network is semantically identical and still functional."*

It doesn't test edge cases in the storage format itself (no schema migrations, no concurrency, no corruption recovery). It's a **behavioral contract test** for the save/load cycle.

## 2. Key Components

### Fixture

```python
@pytest.fixture
def db_path(tmp_path):
    return tmp_path / "test_reasons.db"
```

Leverages pytest's built-in `tmp_path` fixture to give each test an isolated, auto-cleaning temporary directory. The `Storage` instance is created per-test inside each method, so there's no shared state between tests.

### `TestRoundTrip` class

Nine test methods, each following the same skeleton:

| Method | What it verifies |
|--------|-----------------|
| `test_empty_network` | Degenerate case: zero nodes, zero nogoods → empty dicts/lists after load |
| `test_premises` | Node IDs, truth values (`"IN"`), and text survive the round-trip |
| `test_derived_nodes` | `Justification` objects (type, antecedents, label) AND the **derived `dependents` index** are rebuilt on the antecedent nodes |
| `test_retracted_state` | `OUT` truth values are persisted (not just `IN`) |
| `test_nogoods_persisted` | Nogood entries (list of node IDs) survive |
| `test_log_persisted` | The operation log is non-empty and contains an `"add"` action |
| `test_metadata_persisted` | `source`, `source_hash`, and arbitrary `metadata` dict on a node survive |
| `test_propagation_works_after_load` | **Post-load behavior**: retract/restore still triggers correct propagation |
| `test_save_overwrites` | Calling `save()` twice replaces prior state (no append/merge) |

### Contract of `Storage` (as exercised here)

- `Storage(db_path: Path)` — opens/creates a SQLite file at the given path
- `store.save(net: Network)` — serializes the full network (nodes, nogoods, log, metadata) into SQLite; **idempotent overwrite**
- `store.load() -> Network` — deserializes back into a `Network` with nodes, nogoods, log, and rebuilt internal indices
- `store.close()` — releases the DB connection

## 3. Patterns

- **Arrange-Act-Assert per test**: Build a `Network` (arrange) → `save`/`load` (act) → `assert` on the loaded object (assert). No shared setup beyond the fixture.
- **Behavioral, not structural**: The tests don't peek at SQLite tables. They assert on the *Python objects* returned by `load()`. This means the storage format is an implementation detail; if someone swaps SQLite for JSON or Postgres, these tests should still pass.
- **Post-load mutability check** (`test_propagation_works_after_load`): Ensures the loaded network isn't a frozen/detached snapshot — it's a live `Network` whose internal graph (dependents, justifications) is fully wired so inference continues to work.
- **Class grouping without inheritance**: `TestRoundTrip` groups related tests for readability but carries no state or base-class behavior. Purely organizational.

## 4. Dependencies

### Imports (this file → others)

| Import | Role |
|--------|------|
| `reasons.network.Network` | The domain object being persisted |
| `reasons.storage.Storage` | The SUT (System Under Test) |
| `reasons.Justification` | Value object used to build derived nodes in fixtures |
| `pytest` | Test runner, fixture system |
| `tempfile`, `Path` | Unused in the shown code (likely a leftover from an earlier version that used `tempfile.NamedTemporaryFile` before migrating to `tmp_path`) |

### What depends on this file

Nothing in production code imports from `tests/`. This file is consumed only by the test runner (pytest). However, it **indirectly guards** every module that calls `Storage.save()` / `Storage.load()` — that's effectively `reasons/api.py`, `reasons/cli.py`, `reasons/build_wiki.py`, and any long-running service that checkpoints state.

## 5. Flow

Each test follows this sequence:

```
1. Construct Network()
2. Mutate network (add_node, add_nogood, retract, etc.)
3. Storage(db_path).save(net)        # → SQLite INSERT/REPLACE statements
4. store.load()                       # → SELECT * → reconstruct Network
5. Assert on loaded.<field>           # verify semantic equality
6. store.close()                      # release file handle
```

The critical data transformation is step 4: the storage layer must reconstruct not just the raw data (node ID, text, truth value) but also **derived state** — the `dependents` index on antecedent nodes, the linked-list of justifications, and the operation log. `test_derived_nodes` explicitly checks that `loaded.nodes["a"].dependents` contains `"b"` after load, confirming the index is rebuilt, not stored as-is.

## 6. Invariants Enforced

1. **Round-trip fidelity**: `load(save(N)) ≡ N` for all fields the tests touch (nodes, nogoods, log, metadata, truth values).
2. **Index reconstruction**: The `dependents` reverse-index on nodes is *derived* from justifications, not stored independently. After load, it must be consistent.
3. **Overwrite semantics**: A second `save()` on the same path fully replaces prior content (`test_save_overwrites`). No append, no merge.
4. **Post-load liveness**: The loaded `Network` is a fully operational inference engine, not a read-only snapshot.
5. **Truth-value completeness**: Both `IN` and `OUT` states are persisted; loading doesn't default everything to `IN`.

## 7. Error Handling

There is essentially **none** in this file — and that's by design for a unit test:

- No `try/except` blocks. If `store.save()` or `store.load()` raises (e.g., `sqlite3.OperationalError` on a locked file, `KeyError` on a missing node), pytest will report the test as **failed** with the traceback. That's the desired outcome.
- No custom assertions with messages. Plain `assert` statements let pytest show the diff between expected and actual.
- The `db_path` fixture isolates each test in its own temp directory, so a crash in one test can't corrupt another test's DB file.
- `store.close()` is called explicitly at the end of every test. If an assertion *before* `close()` fails, the file handle leaks — but since the file is in a `tmp_path` that gets deleted, this is harmless in practice. (A `try/finally` or context-manager would be more rigorous, but it's a test-file concern, not a production one.)

## Topics to Explore

- [file] `reasons/storage.py` — The SUT itself: understand the SQLite schema, how `save()` batches writes, and how `load()` reconstructs the `dependents` index from justification rows.
- [function] `reasons/network.py:Network.retract` — Called in `test_retracted_state` and `test_propagation_works_after_load`; understanding its cascade semantics (how it propagates `OUT` to dependents) is key to knowing what state the storage layer must preserve.
- [file] `tests/test_pg.py` — A parallel persistence test for the Postgres backend. Comparing the two reveals which round-trip invariants are shared across backends and which are SQLite-specific.
- [repo] `reasons/__init__.py` — Exports `Justification` at the package root; understand the data model (fields, invariants on `antecedents`, the `type` enum) that the storage layer serializes.
- [general] `storage-failure-modes` — What happens in production when the SQLite file is locked, disk is full, or the schema is stale? The tests here don't cover any of these; check if there's retry/migration logic elsewhere.

## Beliefs

- `storage-roundtrip-preserves-truth-values` — Both `"IN"` and `"OUT"` truth values are serialized to and restored from SQLite; loading never defaults a retracted node back to `"IN"`.
- `storage-rebuilds-dependents-index` — The `dependents` reverse-index on nodes is not stored as a separate table; it is recomputed from justification rows during `Storage.load()`, so it is always consistent with the justifications after a round-trip.
- `storage-save-is-overwrite` — Calling `Storage.save()` a second time on the same path fully replaces the prior network state; it does not append or merge nodes/nogoods.
- `test-propagation-validates-liveness` — `test_propagation_works_after_load` asserts that a loaded `Network` supports further inference (retract/restore), meaning the storage layer must reconstruct all internal graph links, not just flat node data.
- `storage-tests-are-backend-agnostic` — The test file asserts on `Network` object attributes, never on SQLite-specific artifacts (table names, row counts), so the storage backend can be swapped without modifying these tests.
