# File: tests/test_api.py

**Date:** 2026-05-08
**Time:** 14:07

## Purpose

`tests/test_api.py` is the primary test suite for `reasons_lib/api.py` — the functional Python API that wraps the TMS (Truth Maintenance System) engine. It validates every public API function through isolated, self-contained tests using ephemeral SQLite databases. This file serves as both a regression suite and a living contract for the API's behavior: what each function accepts, returns, and raises.

## Key Components

### Fixture: `db_path`

A session-scoped fixture (defined at module level and overridden in `TestUpdateNode`) that creates a fresh SQLite database via `api.init_db()` in a pytest `tmp_path`. Every test gets its own database, ensuring complete isolation.

### Test Classes (16 total)

Each class maps 1:1 to an API function:

| Class | API function | What it tests |
|---|---|---|
| `TestInitDb` | `api.init_db()` | Creation, duplicate rejection, force-overwrite |
| `TestAddNode` | `api.add_node()` | Premises, SL-justified derivations, duplicate rejection |
| `TestRetractNode` | `api.retract_node()` | Single retraction, cascade to dependents, missing/already-OUT edge cases |
| `TestAssertNode` | `api.assert_node()` | Restoration with cascade, idempotent assert on IN node |
| `TestPropagate` | `api.propagate()` | No-op propagation, dirty-state repair via direct `Storage` manipulation |
| `TestGetStatus` | `api.get_status()` | Empty DB, node counting |
| `TestShowNode` | `api.show_node()` | Full node details including source, missing-node error |
| `TestExplainNode` | `api.explain_node()` | Premise trace, multi-hop chain trace |
| `TestAddNogood` | `api.add_nogood()` | Nogood registration, ID assignment, side-effect changes |
| `TestGetBeliefSet` | `api.get_belief_set()` | Returns only IN node IDs |
| `TestGetLog` | `api.get_log()` | Log existence, `last=N` truncation |
| `TestExportNetwork` | `api.export_network()` | JSON-serializable network snapshot |
| `TestEndToEnd` | Multiple | Full retract-and-restore cycle on a 3-node chain |
| `TestListNodesDepth` | `api.list_nodes()` | `min_depth`, `max_depth`, and combined range filtering |
| `TestFtsSearch` | `api.search()` / `_fts_search()` | 13 tests covering stemming, relaxation, stop words, depth expansion, query sanitization |
| `TestListGated` | `api.list_gated()` | Gate lifecycle: no gates, active, satisfied, multiple per blocker, superseded exclusion |
| `TestListNegative` | `api.list_negative()` | LLM-based negative-belief classification with mocked `invoke_model` |
| `TestUpdateNode` | `api.update_node()` | Text/source updates, preservation of justifications, updates on OUT nodes |

## Patterns

**One class per API function.** Each test class is named `Test<ApiFunction>` and exhaustively covers the happy path, edge cases, and error paths for that single function.

**Ephemeral databases.** The `db_path` fixture creates a throwaway SQLite file in `tmp_path`. No test shares state with another — this is critical for a stateful system like a TMS where node truth values depend on history.

**Return-value-as-dict.** The API uses a dictionary return convention (`result["changed"]`, `result["truth_value"]`, etc.) rather than domain objects. Tests validate specific keys in these dicts.

**Mock boundary at LLM.** `TestListNegative` mocks `reasons_lib.llm.invoke_model` to test classification logic without calling an actual LLM. It covers parsing robustness: multiline JSON, malformed responses, prose-wrapped JSON, unknown IDs, and batching across 120 candidates.

**Direct storage manipulation for edge cases.** `TestPropagate.test_with_changes` bypasses the API to force a dirty state via `Storage`, then verifies that `api.propagate()` repairs it. This is the only test that breaks the API abstraction — intentionally, to test the repair path.

**Fixture override.** `TestUpdateNode` defines its own `db_path` fixture that pre-populates nodes (`a`, `b`, `derived-ab`), shadowing the module-level fixture to reduce per-test setup.

## Dependencies

**Imports:**
- `reasons_lib.api` — the entire public API surface under test
- `reasons_lib.storage.Storage` — used once in `TestPropagate` to inject dirty state
- `reasons_lib.api._fts_search`, `_fts_query` — private FTS helpers tested directly in `TestFtsSearch`
- `unittest.mock.patch` — mocks `reasons_lib.llm.invoke_model` in `TestListNegative`

**Imported by:** Nothing — this is a leaf test module.

## Flow

Each test follows the same pattern:

1. **Arrange**: Get a fresh `db_path` from the fixture, optionally add nodes to build a dependency graph
2. **Act**: Call the API function under test
3. **Assert**: Check the returned dict for expected keys/values, or verify side effects via follow-up API calls (`show_node`, `get_status`)

The `TestEndToEnd` class chains multiple operations: add 3-node chain → verify all IN → retract root → verify all OUT → assert root → verify all IN. This validates that the retraction/restoration cascade works end-to-end through the API layer.

For `TestFtsSearch`, the flow adds nodes with specific text, then queries with terms that test edge cases of the FTS engine: stemming (`"deactivation"` matching `"auto-deactivated"`), progressive relaxation (dropping terms until something matches), stop-word filtering, and depth-based antecedent expansion.

## Invariants

- **`init_db` refuses to overwrite** unless `force=True` — prevents accidental data loss.
- **Duplicate node IDs raise `ValueError`** — enforced by `add_node`.
- **Missing nodes raise `KeyError`** — enforced by `retract_node`, `show_node`, `update_node`.
- **Retraction cascades are complete** — retracting a root must flip all transitive dependents to OUT.
- **Assertion restoration is symmetric** — asserting a retracted root must restore the same set that was retracted.
- **Idempotent operations return empty `changed`** — retracting an OUT node or asserting an IN node is a no-op.
- **`get_belief_set` returns only IN nodes** — OUT nodes are excluded.
- **Superseded gated nodes are excluded from `list_gated`** — prevents stale gates from appearing.
- **`update_node` preserves justifications and truth value** — editing text doesn't alter the node's logical status.
- **FTS queries with only single-char words return empty** — prevents meaningless matches.
- **`list_negative` batches at most 40 candidates per LLM call** — 120 candidates produce exactly 3 calls.

## Error Handling

The API uses Python exceptions at system boundaries:

- `FileExistsError` from `init_db` when the database already exists
- `ValueError` from `add_node` for duplicate IDs
- `KeyError` from `retract_node`, `show_node`, `update_node` for missing nodes
- `FileNotFoundError` from `list_negative` when the Claude CLI isn't installed — this propagates unhandled (tested in `test_claude_not_found_propagates`)

Malformed LLM responses in `list_negative` are handled gracefully: if JSON parsing fails, the result is `count: 0` rather than an exception. Unknown IDs returned by the LLM are silently filtered out.

## Topics to Explore

- [file] `reasons_lib/api.py` — The implementation behind every assertion in this file; understanding the dict return contracts and the FTS query pipeline
- [function] `reasons_lib/api.py:_fts_search` — The progressive relaxation and stop-word filtering logic that 8 of these tests validate
- [file] `reasons_lib/network.py` — The TMS engine that powers retraction cascades, assertion restoration, and nogood detection
- [function] `reasons_lib/api.py:list_negative` — The LLM-batching, JSON-parsing, and keyword-prefiltering pipeline tested extensively in `TestListNegative`
- [file] `tests/test_network.py` — The lower-level network tests that `TestEndToEnd` explicitly references as covering "the same scenarios"

## Beliefs

- `api-tests-one-class-per-function` — Each test class in `test_api.py` maps to exactly one public `api.*` function, providing exhaustive coverage of that function's contract
- `api-retract-cascade-symmetry` — Retracting a root node and re-asserting it must produce identical `changed` sets, verified by `TestEndToEnd.test_retract_and_restore_chain`
- `list-negative-batches-at-40` — `api.list_negative` splits candidates into batches of 40, so 120 keyword-matching nodes produce exactly 3 LLM calls
- `fts-progressive-relaxation` — When a multi-term FTS query returns no results, the search engine progressively drops terms until it finds matches or exhausts all subsets
- `update-node-preserves-justifications` — `api.update_node` modifies text/source metadata without altering the node's justification list or truth value

