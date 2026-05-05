# File: tests/test_api.py

**Date:** 2026-05-05
**Time:** 15:15

## Purpose

`tests/test_api.py` is the primary test suite for `reasons_lib.api` — the functional Python API that wraps the TMS (Truth Maintenance System) engine. It validates every public API function end-to-end against a real SQLite database (no mocks of the storage layer), ensuring that belief creation, retraction, assertion, propagation, search, gating, and negation classification all behave correctly. It's the contract test: if a consumer calls `api.add_node(...)`, these tests define what they should expect back.

## Key Components

### Fixture: `db_path`

Every test class uses a shared `pytest` fixture that creates a fresh SQLite database in `tmp_path`. This guarantees test isolation — each test gets its own empty TMS.

### Test Classes (one per API function)

| Class | API under test | What it proves |
|---|---|---|
| `TestInitDb` | `api.init_db()` | Creates new DB, refuses overwrite unless `force=True` |
| `TestAddNode` | `api.add_node()` | Premise creation, SL-justified derived nodes, duplicate rejection |
| `TestRetractNode` | `api.retract_node()` | Single retraction, cascade through dependents, missing-node error, idempotent retract |
| `TestAssertNode` | `api.assert_node()` | Restores retracted nodes (with cascade), idempotent assert |
| `TestPropagate` | `api.propagate()` | Re-derives truth values when the network is out of sync |
| `TestGetStatus` | `api.get_status()` | Empty and populated status reports |
| `TestShowNode` | `api.show_node()` | Full node detail retrieval, missing-node error |
| `TestExplainNode` | `api.explain_node()` | Explanation trace for premises and derived chains |
| `TestAddNogood` | `api.add_nogood()` | Contradiction recording and resolution |
| `TestGetBeliefSet` | `api.get_belief_set()` | Returns only IN node IDs |
| `TestGetLog` | `api.get_log()` | Audit log retrieval with `last=N` filtering |
| `TestExportNetwork` | `api.export_network()` | Full network serialization |
| `TestEndToEnd` | Multiple | Full retract-restore-cascade lifecycle |
| `TestListNodesDepth` | `api.list_nodes()` | Depth-based filtering (`min_depth`, `max_depth`) |
| `TestFtsSearch` | `api.search()` / `_fts_search()` | Full-text search with stemming, relaxation, stop-word filtering, depth expansion |
| `TestListGated` | `api.list_gated()` | GATE belief enumeration, blocker tracking, supersession exclusion |
| `TestListNegative` | `api.list_negative()` | LLM-based negative-belief classification with batching |

## Patterns

**One class per API function.** Each public function in `reasons_lib.api` gets its own test class. This makes it trivial to find tests for a given function and to run them in isolation.

**Real database, no storage mocks.** Tests hit actual SQLite through `Storage`. The only mocking is for the LLM layer (`reasons_lib.llm.invoke_model`) in `TestListNegative` and `TestFtsSearch`'s long-query test. This is a deliberate integration-test strategy — the API's contract includes its interaction with SQLite and FTS5.

**`db_path` as a string, not a connection.** The API is designed to accept a `db_path` keyword argument on every call, opening/closing its own connection internally. Tests pass the path as a string, verifying that the API manages its own connection lifecycle correctly.

**Exception-based error signaling.** Missing nodes raise `KeyError`, duplicate nodes raise `ValueError`, existing databases raise `FileExistsError`. Tests assert these with `pytest.raises`.

**Mock-based LLM isolation.** `TestListNegative` patches `reasons_lib.llm.invoke_model` to control LLM responses — testing the parsing, filtering, batching, and error-handling logic without network calls. The mock returns raw JSON strings, simulating what the real LLM would return.

## Dependencies

**Imports:**
- `reasons_lib.api` — the module under test (all public functions)
- `reasons_lib.api._fts_search`, `_fts_query` — internal FTS helpers tested directly in `TestFtsSearch`
- `reasons_lib.storage.Storage` — used once in `TestPropagate.test_with_changes` to manually corrupt truth values and verify propagation repairs them
- `reasons_lib.llm.invoke_model` — mocked in `TestListNegative` to isolate LLM classification
- `unittest.mock.patch`, `pytest` — test infrastructure

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test follows this pattern:

1. **Setup** — The `db_path` fixture creates a fresh DB via `api.init_db()`.
2. **Populate** — Tests call `api.add_node()` to build a belief network (premises, derived nodes, gated nodes).
3. **Act** — The function under test is called (retract, assert, search, etc.).
4. **Assert** — Return values are checked for expected structure and content.

The most complex flow is `TestPropagate.test_with_changes`: it builds a network, retracts and re-asserts a premise, then uses `Storage` directly to force a node's truth value OUT of sync with its justifications, then calls `api.propagate()` and verifies it repairs the inconsistency.

For `TestListNegative`, the flow adds a keyword-filtering stage (pre-LLM) and batching (splitting large candidate sets across multiple LLM calls), both verified by checking `candidates`, `count`, and `mock_llm.call_count`.

## Invariants

- **Retraction cascades are complete.** `test_retract_cascades` and `test_retract_and_restore_chain` assert that retracting a premise changes *all* transitively dependent nodes.
- **Assert/retract are inverses.** The end-to-end test proves `retract → assert` restores the original state exactly.
- **Retract is idempotent.** Retracting an already-OUT node returns `changed == []`.
- **Assert is idempotent.** Asserting an already-IN node returns `changed == []`.
- **Duplicate node IDs are rejected.** `add_node` raises `ValueError` on collision.
- **Missing nodes raise `KeyError`.** Both `retract_node` and `show_node` enforce this.
- **FTS progressive relaxation has bounds.** `test_long_query_does_not_explode` caps `_fts_query` invocations at 51 for a 20-term query.
- **Search depth expansion is precise.** Depth-0 returns only direct hits, depth-1 adds direct antecedents, depth-2 adds transitive antecedents.
- **Gated nodes with retracted blockers don't appear.** `test_satisfied_gate` verifies this.
- **Superseded gated nodes are excluded.** `test_superseded_excluded` ensures `list_gated` respects supersession.
- **LLM batching splits at ~50 candidates per batch.** `test_batching_large_set` (120 candidates) expects exactly 3 LLM calls, confirming a batch size near 50.
- **Malformed LLM responses degrade gracefully.** Returns `count: 0` rather than raising.
- **LLM-returned unknown IDs are silently dropped.** Only IDs present in the DB appear in results.

## Error Handling

The API uses exceptions for caller errors and graceful degradation for LLM failures:

| Scenario | Behavior |
|---|---|
| `init_db` on existing path | `FileExistsError` |
| `add_node` with duplicate ID | `ValueError` |
| `retract_node` / `show_node` on missing ID | `KeyError` |
| LLM returns malformed JSON | `list_negative` returns `count: 0` |
| LLM returns nonexistent node IDs | Silently filtered out |
| LLM binary (`claude`) not found | `FileNotFoundError` propagates to caller |
| Retract already-OUT node | No error, returns `changed: []` |

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The implementation behind every function tested here; understanding the return-value contracts and internal connection management
- [function] `reasons_lib/api.py:list_negative` — The keyword pre-filter, LLM batching logic, and JSON extraction that `TestListNegative` exercises
- [function] `reasons_lib/api.py:_fts_search` — Progressive relaxation algorithm: how it drops terms to broaden FTS5 queries
- [file] `reasons_lib/network.py` — The TMS engine that powers retraction cascades, assertion restoration, and propagation
- [file] `tests/test_network.py` — Lower-level tests for the TMS network directly, which `TestEndToEnd` mirrors through the API layer

## Beliefs

- `api-tests-use-real-sqlite` — All API tests run against a real SQLite database; storage is never mocked, ensuring the API contract includes correct SQL/FTS5 behavior
- `retract-assert-inverse-tested` — The test suite explicitly verifies that retract followed by assert restores the full network state including all transitive dependents
- `list-negative-batches-at-50` — `list_negative` splits candidate nodes into batches of ~50 for LLM classification, verified by the 120-candidate test expecting exactly 3 LLM calls
- `fts-relaxation-bounded` — Progressive FTS query relaxation is bounded: a 20-term query produces at most 51 `_fts_query` invocations
- `api-error-contract-exceptions` — The API signals caller errors via standard Python exceptions (`KeyError`, `ValueError`, `FileExistsError`) and reserves graceful degradation for LLM/external failures

