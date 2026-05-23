# File: tests/test_api.py

**Date:** 2026-05-11
**Time:** 12:56

## Purpose

`tests/test_api.py` is the integration test suite for `reasons_lib.api` — the functional Python API that wraps the TMS (Truth Maintenance System) engine. It validates every public API function end-to-end: creating databases, adding/retracting/asserting nodes, searching, exporting, clustering, deduplication, and negative-belief classification. This is the primary contract test for the API layer — if a behavior works here, downstream consumers (CLI, MCP server) can rely on it.

## Key Components

### Fixture: `db_path`

Most test classes share a common pattern: a `db_path` fixture that creates a fresh SQLite database in `tmp_path` via `api.init_db()`. This gives every test an isolated, empty TMS network. Some classes override it (e.g., `TestUpdateNode` pre-populates nodes; `TestListClusters` uses `db_with_beliefs`).

### Test Classes (by API function)

| Class | API under test | What it validates |
|---|---|---|
| `TestInitDb` | `api.init_db()` | Creation, duplicate rejection, force-overwrite |
| `TestAddNode` | `api.add_node()` | Premises, SL-justified derived nodes, duplicate rejection |
| `TestRetractNode` | `api.retract_node()` | Single retraction, cascade through dependents, missing-node error, idempotent re-retract |
| `TestAssertNode` | `api.assert_node()` | Restoration cascades, idempotent re-assert |
| `TestPropagate` | `api.propagate()` | No-op case, and forcing re-evaluation after manually corrupting truth values via `Storage` |
| `TestGetStatus` | `api.get_status()` | Empty and populated network summary |
| `TestShowNode` | `api.show_node()` | Full node detail retrieval, missing-node error |
| `TestExplainNode` | `api.explain_node()` | Premise explanation, multi-hop justification chains |
| `TestAddNogood` | `api.add_nogood()` | Contradiction registration and side effects |
| `TestGetBeliefSet` | `api.get_belief_set()` | Returns only IN node IDs |
| `TestGetLog` | `api.get_log()` | Audit log retrieval with `last=N` filtering |
| `TestExportNetwork` | `api.export_network()` | JSON-serializable snapshot of the full network |
| `TestEndToEnd` | Multiple | Full retract-and-restore lifecycle on a 3-node chain |
| `TestListNodesDepth` | `api.list_nodes()` | `min_depth`/`max_depth` filtering by justification depth |
| `TestFtsSearch` | `api.search()`, `_fts_search()` | Porter stemming, progressive relaxation, stop-word filtering, depth-based antecedent expansion, punctuation handling, long-query bounds |
| `TestListGated` | `api.list_gated()` | GATE belief enumeration, satisfied gates, superseded exclusion, blocker text |
| `TestListNegative` | `api.list_negative()` | LLM-based negative-belief classification with mocked `invoke_model` |
| `TestUpdateNode` | `api.update_node()` | Text/source mutation, justification preservation, OUT-node updates |
| `TestListClusters` | `api.list_clusters()` | Status filtering, empty network, seed/n_clusters passthrough |
| `TestDeduplicateSemantic` | `api.deduplicate()` | Semantic similarity detection, auto-retraction, threshold filtering |

## Patterns

**Temp-DB isolation**: Every test gets its own SQLite file via `tmp_path`. No shared state between tests — this is critical because TMS operations have side effects (cascading retractions).

**Mock boundary at LLM**: Tests that involve LLM calls (`TestListNegative`, `TestListClusters`) mock `reasons_lib.llm.invoke_model` or `reasons_lib.cluster.list_clusters`. The API is tested; the LLM is not. This keeps tests fast and deterministic.

**Progressive complexity**: Each class starts with the simplest case (empty DB, single node) and builds up to edge cases (cascading chains, malformed LLM responses, long queries).

**Conditional skip**: `TestDeduplicateSemantic` is gated behind `HAS_CLUSTER_DEPS` — it only runs if `sentence-transformers` and `scikit-learn` are installed. This avoids hard failures in minimal environments.

**Direct Storage manipulation**: `TestPropagate.test_with_changes` bypasses the API to corrupt a node's truth value via `Storage`, then verifies `propagate()` fixes it. This is the only test that touches internals directly, and it's testing the "self-healing" propagation path.

## Dependencies

**Imports from the project:**
- `reasons_lib.api` — the entire public API surface (primary target)
- `reasons_lib.storage.Storage` — used once in `TestPropagate` to simulate corruption
- `reasons_lib.api._fts_search`, `_fts_query` — private FTS helpers tested directly for search behavior
- `reasons_lib.cluster.HAS_CLUSTER_DEPS` — feature flag for optional dependency

**External:**
- `pytest` — test framework and fixtures
- `unittest.mock.patch` — mocking LLM and cluster calls

**Nothing imports this file** — it's a test module, consumed only by pytest.

## Flow

A typical test follows this pattern:

1. `db_path` fixture creates a fresh SQLite DB via `api.init_db()`
2. Test adds nodes via `api.add_node()`, building a small network
3. Test performs the operation under test (retract, search, export, etc.)
4. Test asserts on the returned dict structure

For LLM-dependent tests, the flow adds a mock layer:
1. Build network
2. `patch("reasons_lib.llm.invoke_model")` with a canned response
3. Call the API
4. Assert on results AND verify mock call counts/arguments

## Invariants

- **`add_node` with duplicate ID raises `ValueError`** — the network enforces unique node IDs.
- **`retract_node` / `show_node` with missing ID raises `KeyError`** — no silent failures for nonexistent nodes.
- **Retraction cascades**: retracting a premise also retracts all transitively dependent derived nodes. Restoration reverses this.
- **Idempotent retract/assert**: re-retracting an OUT node or re-asserting an IN node returns `changed == []`.
- **`init_db` refuses to overwrite** unless `force=True` — prevents accidental data loss.
- **FTS long-query bound**: `_fts_search` makes at most 51 calls to `_fts_query` regardless of query length (tested explicitly with a 20-term query).
- **`list_negative` batches to 3 LLM calls** for 120 candidates (batch size ~40-50).
- **`list_gated` excludes superseded beliefs** — once a GATE belief is superseded, it no longer appears in blocker reports.
- **Search depth expansion**: `depth=0` returns only direct matches; `depth=1` adds direct antecedents; `depth=2` adds transitive antecedents.

## Error Handling

The API uses standard Python exceptions as its error protocol:
- **`FileExistsError`** from `init_db()` when the DB already exists
- **`ValueError`** from `add_node()` on duplicate IDs
- **`KeyError`** from `retract_node()`, `show_node()`, `update_node()` on missing nodes
- **`FileNotFoundError`** from `list_negative()` when the `claude` CLI is missing — this propagates up unhandled (tested in `test_claude_not_found_propagates`)

Malformed LLM responses are handled gracefully — `list_negative` returns `count: 0` if the LLM returns non-JSON, and filters out unknown IDs from otherwise valid responses.

## Topics to Explore

- [file] `reasons_lib/api.py` — The implementation behind every function tested here; understanding the dict return contracts is essential
- [function] `reasons_lib/api.py:_fts_search` — The progressive relaxation and stop-word filtering logic that the FTS tests exercise
- [file] `reasons_lib/network.py` — The TMS engine that powers retraction cascades, SL justifications, and outlist/GATE semantics
- [function] `reasons_lib/api.py:list_negative` — The LLM-based negative belief classifier, including batching strategy and response parsing
- [file] `reasons_lib/cluster.py` — Semantic deduplication and clustering implementation behind `deduplicate()` and `list_clusters()`

## Beliefs

- `api-retract-cascade-tested` — Retracting a premise cascades to all transitively dependent derived nodes, and restoring it reverses the cascade (verified by `TestEndToEnd` and `TestRetractNode`)
- `api-fts-search-bounds-query-calls` — `_fts_search` makes at most 51 internal `_fts_query` calls regardless of input query length, preventing combinatorial explosion
- `api-list-negative-graceful-on-bad-llm` — `list_negative` returns `count: 0` when the LLM returns non-JSON or unknown IDs, rather than raising
- `api-init-db-refuses-overwrite` — `init_db` raises `FileExistsError` if the database already exists unless `force=True` is passed
- `api-list-gated-excludes-superseded` — `list_gated` filters out GATE beliefs that have been superseded, so they don't appear as active blockers

