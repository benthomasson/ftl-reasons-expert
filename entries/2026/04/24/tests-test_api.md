# File: tests/test_api.py

**Date:** 2026-04-24
**Time:** 16:51

# `tests/test_api.py` — Explained

## 1. Purpose

This file is the **integration test suite for `reasons_lib/api.py`** — the functional Python API that sits between the CLI/external callers and the core `Network` + `Storage` layer. Its job is to verify that every public API function:

- Opens a fresh SQLite database, performs the operation, persists correctly, and returns the right dict shape.
- Raises the correct exceptions for invalid inputs.
- Produces correct cascading side effects (retraction cascades, restoration).

It does **not** test the lower-level `Network` propagation logic directly (that's `test_network.py`). Instead, it tests the same scenarios through the API's open-operate-save-close cycle, which exercises the `_with_network` context manager and SQLite round-tripping on every call.

## 2. Key Components

### Fixture: `db_path`
Creates a temporary SQLite database per test via `tmp_path`, calls `api.init_db()`, and returns the path string. Every test gets an isolated, empty database — no cross-test contamination.

### Test classes (one per API function):

| Class | Tests | What it covers |
|---|---|---|
| `TestInitDb` | 3 | Create new DB, refuse existing, force-overwrite |
| `TestAddNode` | 3 | Premise creation, SL-justified derived node, duplicate rejection |
| `TestRetractNode` | 4 | Single retraction, cascade to dependents, missing-node error, idempotent retraction |
| `TestAssertNode` | 2 | Restoration cascade, idempotent assert |
| `TestGetStatus` | 2 | Empty DB, populated DB counts |
| `TestShowNode` | 2 | Full node detail retrieval, missing-node error |
| `TestExplainNode` | 2 | Premise explanation, chain explanation |
| `TestAddNogood` | 1 | Contradiction recording + backtracking |
| `TestGetBeliefSet` | 1 | Only IN nodes returned |
| `TestGetLog` | 2 | Full log, `last=N` truncation |
| `TestExportNetwork` | 1 | JSON export structure |
| `TestEndToEnd` | 1 | Full retract → restore → verify cycle across a 3-node chain |
| `TestListNodesDepth` | 3 | Depth-based filtering: `min_depth`, `max_depth`, range |

## 3. Patterns

- **One class per API function** — mirrors the API surface directly. Easy to find the test for any API function.
- **Fixture-based isolation**: `tmp_path` + `db_path` fixture means every test hits a fresh SQLite file. No shared state, no teardown needed.
- **Black-box testing**: Tests call `api.*` functions and assert on returned dicts. They never reach into `Network` or `Storage` internals.
- **Cascade verification via sets**: `set(result["changed"]) == {"a", "b", "c"}` — order-independent comparison for BFS propagation results.
- **Exception contracts tested with `pytest.raises`**: `FileExistsError`, `ValueError`, `KeyError` — each maps to a specific API precondition violation.

## 4. Dependencies

**Imports:**
- `pytest` — test framework + fixtures
- `reasons_lib.api` — the module under test

**Imported by:** Nothing — this is a leaf test file.

**Transitive exercise:** Each `api.*` call goes through `api._with_network` → `Storage.load()` / `Storage.save()` → `Network.*`, so these tests implicitly exercise the full stack from API → network → SQLite.

## 5. Flow

A typical test follows this pattern:

1. `db_path` fixture creates an empty database.
2. Test builds up state by calling `api.add_node()` one or more times (premises, then derived nodes).
3. Test performs the operation under test (retract, assert, explain, etc.).
4. Test asserts on the returned dict's keys and values.

The `TestEndToEnd` class shows the full lifecycle: add 3-node chain → verify all IN → retract root → verify cascade → assert root → verify restoration. This mirrors the exact workflow the CLI exposes.

## 6. Invariants Enforced

- **`init_db` refuses to overwrite** unless `force=True` — prevents accidental data loss.
- **Duplicate node IDs raise `ValueError`** — node IDs are unique within a network.
- **Operations on missing nodes raise `KeyError`** — `retract_node`, `show_node` validate existence.
- **Retraction cascades are complete**: retracting a premise makes all transitively-dependent derived nodes go OUT.
- **Restoration cascades are symmetric**: asserting a retracted premise restores all dependents whose justifications become valid again.
- **Idempotent operations return empty `changed` lists** — retracting an already-OUT node or asserting an already-IN node is a no-op.
- **Depth filtering**: `min_depth=1` excludes premises (depth 0), `max_depth=0` excludes derived nodes.

## 7. Error Handling

The tests verify three exception types from the API:

| Exception | Condition | Test |
|---|---|---|
| `FileExistsError` | `init_db` on existing path without `force` | `TestInitDb.test_refuses_existing` |
| `ValueError` | `add_node` with a duplicate ID | `TestAddNode.test_add_duplicate_raises` |
| `KeyError` | `retract_node` or `show_node` on a nonexistent ID | `TestRetractNode.test_retract_missing_raises`, `TestShowNode.test_show_missing_raises` |

No exceptions are swallowed — the API is designed to raise on invalid input and let the caller (CLI, LangGraph tool) handle display.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The module under test; the `_with_network` context manager and dict-return contracts are the API's architectural spine
- [file] `tests/test_network.py` — Tests the same propagation scenarios at the `Network` layer without SQLite; compare to understand which behaviors are tested at each level
- [function] `reasons_lib/api.py:_with_network` — The open/operate/save/close pattern that every API function uses; understanding it explains why these tests exercise persistence implicitly
- [file] `tests/test_outlist.py` — Non-monotonic reasoning (outlist/unless) is not covered in `test_api.py` but is a core TMS feature worth understanding
- [general] `cascade-symmetry` — The relationship between retraction cascades and restoration cascades is the central invariant of Doyle's TMS; `TestEndToEnd` and `TestAssertNode` are the tests that verify it end-to-end

## Beliefs

- `api-tests-black-box` — All tests in `test_api.py` call only public `api.*` functions and assert on returned dicts; no internal `Network` or `Storage` state is inspected directly
- `api-tests-isolated-db` — Every test gets its own temporary SQLite database via the `db_path` fixture; tests share no state and can run in any order
- `api-cascade-symmetry-tested` — `TestEndToEnd.test_retract_and_restore_chain` verifies that retracting a root premise cascades OUT all 3 nodes and re-asserting it restores all 3, confirming symmetric cascade behavior through the full API+Storage stack
- `api-idempotent-retract-assert` — Retracting an already-OUT node and asserting an already-IN node both return `changed == []`, tested explicitly in `TestRetractNode.test_retract_already_out` and `TestAssertNode.test_assert_already_in`
- `api-tests-cover-subset` — `test_api.py` covers ~12 of the ~30 public API functions; features like challenge/defend, namespaces, access_tags, search, compact, and deduplication have no coverage in this file

