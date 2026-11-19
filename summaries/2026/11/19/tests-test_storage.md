---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_storage.py

# `tests/test_storage.py`

## Purpose

This is a pytest module that verifies SQLite persistence for the `reasons` package. Its job is to prove that a `Network` can be saved through `Storage`, loaded back into a new `Network`, and still behave like the original network.

It is a round-trip integration test file: it builds network state through the public `Network` API, persists it, reloads it, and asserts that nodes, justifications, truth values, nogoods, logs, metadata, and post-load propagation all survive.

A notable repository detail: this test imports from `reasons`, not from `reasons_lib`. So it exercises `reasons/storage.py` and `reasons/network.py`, not the duplicate `reasons_lib` package.

## Key Components

### Fixture: `db_path(tmp_path)`

```python
@pytest.fixture
def db_path(tmp_path):
    return tmp_path / "test_reasons.db"
```

- Uses pytest’s built-in `tmp_path` fixture.
- Returns a unique `.db` path per test.
- Provides isolation so tests do not share SQLite state.
- The imported `tempfile` and `Path` modules are not used in this file.

### Test class: `TestRoundTrip`

Groups all persistence round-trip tests under one class. Each test follows the same pattern:

1. Build a `Network`.
2. Instantiate `Storage(db_path)`.
3. Call `store.save(net)`.
4. Call `store.load()`.
5. Assert on the loaded `Network`.
6. Call `store.close()`.

The tests assume the following `Storage` contract:

- `Storage(path)` accepts a filesystem path.
- `save(network)` persists a `Network` snapshot.
- `load()` returns a reconstructed `Network`.
- `close()` releases the underlying SQLite resource.

They also assume the following `Network` API:

- `Network()`
- `add_node(node_id, text, justifications=..., source=..., source_hash=..., metadata=...)`
- `retract(node_id)`
- `assert_node(node_id)`
- `add_nogood([node_ids])`
- Attributes: `nodes`, `nogoods`, `log`

And the following `Justification` contract:

- `Justification(type=..., antecedents=[...], label=...)`
- Attributes: `type`, `antecedents`, `label`

### Test methods

| Test | What it verifies |
|---|---|
| `test_empty_network` | An empty network round-trips as `loaded.nodes == {}` and `loaded.nogoods == []`. |
| `test_premises` | Premise nodes retain their IDs, text, and default `truth_value == "IN"`. |
| `test_derived_nodes` | Derived nodes retain justifications. Also verifies that the dependent index is rebuilt: `"b" in loaded.nodes["a"].dependents`. |
| `test_retracted_state` | Retracted truth values persist. After `net.retract("a")`, both `a` and dependent `b` load as `"OUT"`. |
| `test_nogoods_persisted` | A nogood added with `net.add_nogood(["a", "d"])` survives save/load with exact node order preserved. |
| `test_log_persisted` | The network log is persisted. The loaded log is non-empty and contains an entry with `"action": "add"`. |
| `test_metadata_persisted` | Node `source`, `source_hash`, and `metadata` dictionaries round-trip exactly. |
| `test_propagation_works_after_load` | The loaded network is operationally valid: `retract` and `assert_node` still propagate through dependencies after loading. |
| `test_save_overwrites` | Saves `a`, adds `b`, saves again, then loads and expects keys `{"a", "b"}`. Despite the name, it does not explicitly test deletion of a removed node; it verifies that the second save still contains the full latest snapshot. |

## Patterns

- **Round-trip testing**: Build in memory, persist, reload, assert equivalent state.
- **Black-box integration testing**: The tests use public `Network` and `Storage` methods rather than inspecting SQLite tables directly.
- **Pytest fixture injection**: `db_path` is created from `tmp_path`, giving each test an isolated database file.
- **Arrange-Act-Assert**: Each test follows the standard three-phase pattern.
- **Test-class grouping**: All persistence round-trip cases live in `TestRoundTrip`.
- **Manual resource cleanup**: Each test explicitly calls `store.close()` rather than using a context manager or `try/finally`.

## Dependencies

### Imports used

- `pytest` for fixtures and test discovery.
- `reasons.Justification` for constructing justifications in derived-node tests.
- `reasons.network.Network` for building and reasoning about graphs.
- `reasons.storage.Storage` for SQLite persistence.

### Imports unused

- `tempfile`
- `pathlib.Path`

These are imported but not referenced in the file.

### What imports this file

This file is a pytest test module. It is normally discovered and executed by the test runner rather than imported by production code.

## Flow

Execution is straightforward:

1. Pytest discovers `TestRoundTrip` and each `test_*` method.
2. For each test, pytest resolves the `db_path` fixture.
3. `tmp_path` provides a unique temporary directory.
4. `db_path` returns `<tmp_path>/test_reasons.db`.
5. The test constructs a `Network`.
6. The test constructs `Storage(db_path)`, which opens/creates the SQLite database.
7. `store.save(net)` serializes the current network state.
8. `store.load()` reconstructs and returns a new `Network`.
9. Assertions compare the loaded network to expected state.
10. `store.close()` closes the SQLite storage.

The data transformation being tested is:

```text
Network object -> Storage.save -> SQLite rows
SQLite rows -> Storage.load -> reconstructed Network object
```

The most important transformation is in `test_derived_nodes`: justifications are persisted and loaded, then the dependent index is rebuilt so that `loaded.nodes["a"].dependents` contains `"b"`. The post-load propagation test then confirms that this rebuilt graph is not just data-compatible but behaviorally compatible.

## Invariants

- Each test uses a unique SQLite path via `tmp_path`.
- Saving and loading must preserve node IDs and node text.
- Loaded premises must have their persisted truth value, with default premise state being `"IN"` in these tests.
- Derived nodes must preserve their `Justification` objects, including `type`, `antecedents`, and `label`.
- The `dependents` index must be rebuilt on load from justification antecedents.
- Retracted `"OUT"` states must persist across save/load.
- After loading, propagation must still work: retracting and re-asserting a premise must update dependent nodes.
- Nogoods must survive save/load, including exact node order in the asserted case.
- Log entries must survive save/load, and at least one `"action": "add"` entry must be present after adding a node.
- Node `source`, `source_hash`, and `metadata` must round-trip exactly.
- Saving the same network twice should produce a loadable latest snapshot containing both existing and newly-added nodes.
- Tests assume `store.close()` is called after use.

## Error Handling

This file has no explicit error handling:

- It does not use `try/except`.
- It does not use `pytest.raises`.
- It does not test invalid paths, corrupted databases, schema migrations, duplicate node IDs, transaction failures, or concurrent access.
- Any exception from `Storage`, `Network`, or `Justification` will propagate and fail the test.
- Any mismatch will raise a normal `AssertionError` from the `assert` statements.
- `store.close()` is called at the end of each test, but not inside a `finally` block or context manager. If an assertion or storage call raises before `close()`, the test code will not close the storage explicitly. In practice, pytest cleanup and garbage collection may handle the temporary files/database, but the connection is not deterministically closed by this file on failure.

## Topics to Explore

- [file] `reasons/storage.py` — Implements the SQLite schema and the `Storage.save`/`Storage.load` behavior exercised by these tests.
- [function] `reasons/storage.py:Storage.save` — Shows how nodes, justifications, truth values, nogoods, logs, and metadata are serialized and whether old rows are cleared.
- [function] `reasons/storage.py:Storage.load` — Shows how SQLite rows are reconstructed into a `Network` and how the dependent index is rebuilt.
- [file] `reasons/network.py` — Defines `add_node`, `retract`, `assert_node`, `add_nogood`, dependency tracking, and propagation semantics assumed here.
- [general] `sqlite-schema-migrations` — Worth checking because this test only covers fresh-database round trips, not schema upgrades or backward compatibility.

## Beliefs

- `storage-roundtrip-preserves-network-state` — Saving and loading through `Storage` preserves nodes, truth values, justifications, nogoods, logs, and node metadata.
- `storage-load-rebuilds-dependents-index` — The tests assert that a loaded network rebuilds `dependents` such that `"b" in loaded.nodes["a"].dependents` after loading a derived node `b`.
- `loaded-networks-remain-operational` — A `Network` loaded from SQLite supports `retract` and `assert_node` with propagation to dependent nodes.
- `storage-tests-target-reasons-package` — This test file imports from `reasons`, not `reasons_lib`, so it does not directly cover the duplicate `reasons_lib/storage.py` implementation.
- `storage-tests-manually-close-connections` — Each test calls `store.close()` after assertions, but not inside `try/finally` or a context manager.
