---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_api.py

# `tests/test_api.py` — Functional Python API Test Suite

## 1. Purpose

This file is the **primary integration test suite** for the `reasons.api` module — the public, dict-in/dict-out functional API that wraps the defeasible reasoning engine. It owns the responsibility of verifying that every function exposed through `reasons.api` behaves correctly against a real SQLite-backed database (via `tmp_path`), covering:

- Core node lifecycle (add, retract, assert, propagate)
- Querying and listing (status, show, list, search)
- The proposal state machine (propose → accept/reject/withdraw)
- Semantic deduplication and clustering
- FTS-based search with stemming, progressive relaxation, and regex
- Tag/source management and filtering
- Export/import round-trip fidelity (timestamps, tags, sources)
- LLM-assisted operations (`list_negative`, `store_llm_proposals`) with mocked model calls

It sits alongside `test_network.py` (which tests the lower-level network primitives) and serves as the **contract test** for anything that calls `reasons.api` — CLI tools, MCP server, HuggingFace integration, or third-party scripts.

## 2. Key Components

### Shared Fixture

```python
@pytest.fixture
def db_path(tmp_path):
    p = str(tmp_path / "test_reasons.db")
    api.init_db(db_path=p)
    return p
```

Every test class (except those that define their own) gets a **fresh, initialized SQLite database** in a pytest-managed temp directory. This guarantees zero cross-test contamination.

### Test Classes by API Surface

| Class | What it verifies |
|---|---|
| `TestInitDb` | `init_db` creation, `FileExistsError` on re-init, `force=True` overwrite |
| `TestAddNode` | Premise/SL creation, duplicate rejection |
| `TestRetractNode` | Single retract, **cascade** to dependents, missing node, idempotent re-retract |
| `TestAssertNode` | Restore after retract, no-op if already IN |
| `TestPropagate` | No-op on clean state; recovers a manually-corrupted OUT node back to IN |
| `TestGetStatus` | Empty network; node listing with counts |
| `TestShowNode` | Field-level assertions (id, text, source, justifications, dependents) |
| `TestExplainNode` | Premise trace, chain trace, **circular dependency detection**, diamond-shaped DAG (no false circular) |
| `TestAddNogood` | Nogood creation, deterministic ID (`nogood-a-b`), cascade effect |
| `TestGetBeliefSet` | Only IN nodes returned |
| `TestGetLog` | Audit log entries; `last=N` pagination |
| `TestExportNetwork` | Full network serialization |
| `TestEndToEnd` | 3-node chain: retract cascades all, assert restores all |
| `TestListNodesDepth` | `min_depth`/`max_depth` filtering, label filtering, premise exclusion from label queries |
| `TestFtsSearch` | Porter stemming, progressive relaxation, stop-word handling, single-char guard, natural-language queries, punctuation, query-length cap (`≤51` FTS calls), depth expansion (0/1/2), `format="compact"` |
| `TestListGated` | `unless=` gate semantics: active, satisfied, multiple-gated-per-blocker, superseded exclusion, blocker text exposure |
| `TestListNegative` | LLM-assisted negative belief detection with **13 sub-scenarios** (empty DB, no matches, classification, LLM filtering, multiline JSON, malformed response, unknown IDs, prose-before-JSON, missing CLI, `visible_to` access tags, single-batch call count, 120-node batching, expanded terms, false-positive exclusion, `skip_llm`) |
| `TestSupersedeWithText` | In-place text replacement, custom ID, access-tag copy, no-tag no-metadata |
| `TestWhatIfSupersede` | **Dry-run** supersede: cascade preview, `already_out`, `does_not_mutate_db`, auto-ID increment (`a-v2` → `a-v3`) |
| `TestSetMetadata` | Key-value set, overwrite, missing node → `KeyError` |
| `TestListClusters` | Status filtering (IN/OUT), seed/n_clusters passthrough, empty network |
| `TestDeduplicateSemantic` | (Gated on `HAS_CLUSTER_DEPS`) Semantic similarity clustering, threshold sensitivity, auto-retract, Jaccard fallback |
| `TestLifecycleTimestamps` | `created_at`, `updated_at`, `retracted_at` semantics; cascade does **not** stamp dependents; assert clears `retracted_at`; export/import round-trip preserves all five timestamp fields |
| `TestProposeRetraction` | Proposal creation, no truth change, auto-stale of older pending, listing, filtering, ID increment |
| `TestProposeSupersession` | Same pattern for supersede; `proposed_text` in snapshot; different actions don't stale each other |
| `TestProposeAddition` | New node proposal; justification/label/source in snapshot; auto-stale |
| `TestAcceptProposal` | Accept retraction (cascade), supersession, addition; drift detection (target already OUT, node deleted, ID collision) → `applied=False`, `status="stale"` |
| `TestRejectProposal` | Reject with reason; no truth change; already-resolved → `ValueError` |
| `TestWithdrawProposal` | Self-withdraw; already-resolved → `ValueError` |
| `TestProposeNogood` | 2+ node requirement, missing node, accept cascades, accept with deleted node → stale |
| `TestProposalTags` | Tags on all four proposal types; sorted; list filtering by tag |
| `TestStoreLlmProposals` | Batch ingestion of LLM output; `update`→`supersede` mapping; skip rules (no text, missing node); mixed batch; pending status |
| `TestOrSearch` | Pipe-delimited OR semantics in FTS |
| `TestRegexSearch` | `regex=True` pattern matching, case-insensitivity, invalid pattern → `ValueError`, alternation |
| `TestNamesOnlyFormat` | `format="names-only"` for search/compact/lookup; budget; OUT exclusion |
| `TestSearchPagination` | `limit`, `offset`, combined |
| `TestStatusPagination` | `limit`/`offset` on `get_status` |
| `TestNodeTags` | Add/remove/get tags; idempotency; survival across retract/assert; `access_tags` migration |
| `TestNodeSources` | Add source(s), multiple sources, survival, no duplication, `show_node` inclusion |
| `TestExportImportTagsSources` | Tags + sources in export; full round-trip preserves source_type, label |
| `TestTagFiltering` | `list_nodes(tag={...})` with topic/status/origin/priority/stream filters; AND across namespaces, OR within; `search` with tag filter |

## 3. Patterns

- **Fixture-per-class isolation**: Every test class gets a fresh DB via `tmp_path`. Classes that need pre-populated state (e.g., `TestWhatIfSupersede`, `TestProposeRetraction`) override the fixture to seed a small network before the test body runs.

- **Dict-based API assertions**: The API returns plain dicts (not dataclasses), so tests assert on keys directly: `result["truth_value"] == "IN"`, `set(result["changed"]) == {"a", "b"}`. This mirrors the JSON-serializable contract the CLI/MCP server relies on.

- **Mock-based LLM testing**: All tests that touch `reasons.llm.invoke_model` use `unittest.mock.patch` to return canned JSON strings. This keeps tests deterministic and eliminates network/model dependency. The `TestListNegative` class is the densest example — 15+ sub-tests covering every edge of the LLM response parsing pipeline.

- **Conditional skip for optional deps**: `TestDeduplicateSemantic` is guarded by `@skip_no_cluster`, which checks `HAS_CLUSTER_DEPS` from `reasons.cluster`. This lets CI run the full suite even without `sentence-transformers`/`scikit-learn` installed.

- **Drift detection as a first-class test**: `TestAcceptProposal` has dedicated tests (`test_drift_target_already_out`, `test_drift_target_deleted`, `test_drift_add_node_already_exists`, `test_drift_supersede_new_id_collision`) that verify the system gracefully marks proposals as `stale` when the world changed between propose and accept.

- **End-to-end chain tests**: `TestEndToEnd` and `TestWhatIfSupersede` exercise multi-node cascades to catch propagation bugs that single-node tests would miss.

- **Private function testing where necessary**: `TestFtsSearch` imports and tests `_fts_search` and `_fts_query` directly (e.g., `test_long_query_does_not_explode` counts internal call invocations), which is a pragmatic concession to verify internal invariants that are hard to observe through the public API alone.

## 4. Dependencies

### Imports (what this file depends on)

| Import | Role |
|---|---|
| `reasons.api` | The module under test — every public function |
| `reasons.storage.Storage` | Used in 3 places to directly mutate network state (e.g., force a node to OUT to test `propagate`) |
| `reasons.llm.invoke_model` | Patched in `TestListNegative` and `TestStoreLlmProposals` tests |
| `reasons.cluster.HAS_CLUSTER_DEPS` | Feature flag for optional ML dependencies |
| `reasons.cluster.list_clusters` | Patched in `TestListClusters` to avoid real embedding computation |
| `pytest` | Test runner, fixtures, parametrize, skip |
| `unittest.mock.patch` | LLM and cluster mocking |
| `json`, `datetime` | Export/import round-trip tests, timestamp assertions |

### What depends on this file (imported_by)

Nothing imports `tests/test_api.py` directly — it's a leaf test module. However, it **protects** the contract that `reasons/cli.py`, `reasons/mcp_server.py`, `reasons/hf.py`, and any external agent scripts rely on when they call `reasons.api.*`.

## 5. Flow

Each test follows the same shape:

```
1. (fixture) Create fresh DB via api.init_db(db_path=tmp)
2. (fixture, if class-level) Seed a small network (2-3 nodes)
3. Act:  Call one api.* function
4. Assert: Check dict keys, truth values, exception types, side-effects
```

For **cascade tests** the flow extends:

```
add a → add b(sl=a) → add c(sl=b)
retract a → assert changed == {a, b, c} → assert status.in_count == 0
assert a  → assert changed == {a, b, c} → assert status.in_count == 3
```

For **LLM-dependent tests** the flow is:

```
Seed nodes with keyword-triggering text ("bug", "error", "critical")
Patch invoke_model to return a specific JSON string
Call api.list_negative(db_path=db)
Assert on result["count"], result["negative"], result["candidates"], result["total"]
Assert mock was/wasn't called, and (sometimes) inspect the prompt
```

The `TestLifecycleTimestamps` class adds a temporal dimension:

```
add node → assert created_at == updated_at
set_metadata → assert updated_at >= previous updated_at
retract → assert retracted_at != ""
assert → assert retracted_at == "" and updated_at >= retracted updated_at
export → import into new DB → assert all 5 timestamp fields match
```

## 6. Invariants

These are the rules the tests **enforce** and that the implementation must guarantee:

1. **Truth-value consistency**: A node is always either `"IN"` or `"OUT"`. Retract sets OUT; assert sets IN. No third state exists.

2. **Cascade completeness**: Retraacting a node must retract **all** transitive dependents (via SL links). Asserting it back must restore all of them. The `changed` set in the result must be exactly the affected set.

3. **Idempotency of terminal operations**: Retracting an already-OUT node returns `changed == []` without raising. Asserting an already-IN node likewise.

4. **Proposal state machine**: A proposal transitions `pending → {accepted, rejected, withdrawn, stale}`. Once resolved, any further accept/reject/withdraw raises `ValueError("not pending")`. A new proposal on the same target auto-stales the previous pending one **of the same action type** (retract doesn't stale supersede, and vice versa).

5. **Drift safety on accept**: If the target's state changed between propose and accept (already OUT, deleted, ID collision), `accept_proposal` returns `applied=False` and marks the proposal `stale` rather than raising.

6. **Timestamp ordering**: `created_at == updated_at` at creation. `updated_at` is monotonically non-decreasing. `retracted_at` is set on explicit retract but **not** on cascade-induced OUT. Assert clears `retracted_at`.

7. **FTS query boundedness**: The internal `_fts_query` is called at most 51 times per search (the `test_long_query_does_not_explode` invariant), preventing combinatorial explosion on long multi-term queries.

8. **Tag immutability across truth changes**: Tags and sources survive retract → assert cycles. They are metadata, not state.

9. **Export/import fidelity**: A full export → JSON → import cycle preserves node text, truth_value, all 5 timestamp fields, tags, sources (including `source_type` and `label`), and SL/justification links.

10. **Nogood minimum size**: A nogood requires at least 2 nodes (`ValueError` otherwise).

## 7. Error Handling

The tests codify a precise **exception taxonomy**:

| Condition | Exception | Test |
|---|---|---|
| `init_db` on existing file (no force) | `FileExistsError` | `TestInitDb.test_refuses_existing` |
| `add_node` with duplicate ID | `ValueError` | `TestAddNode.test_add_duplicate_raises` |
| `retract_node` / `show_node` / `explain_node` on missing ID | `KeyError` | Multiple classes |
| `set_metadata` on missing node | `KeyError` | `TestSetMetadata.test_nonexistent_raises` |
| `propose_retraction` / `propose_supersession` on already-OUT target | `ValueError("already OUT")` | `TestProposeRetraction`, `TestProposeSupersession` |
| `propose_supersession` with `new_id` that exists | `ValueError("already exists")` | `TestProposeSupersession.test_new_id_already_exists` |
| `propose_addition` with existing node ID | `ValueError("already exists")` | `TestProposeAddition.test_node_already_exists` |
| `propose_nogood` with < 2 nodes | `ValueError("at least 2")` | `TestProposeNogood.test_requires_two_nodes` |
| `propose_nogood` with missing node | `KeyError("not found")` | `TestProposeNogood.test_missing_node` |
| `accept` / `reject` / `withdraw` on missing proposal ID | `KeyError` | Multiple proposal classes |
| `accept` / `reject` / `withdraw` on already-resolved proposal | `ValueError("not pending")` | Multiple proposal classes |
| `search` with invalid regex | `ValueError` | `TestRegexSearch.test_regex_invalid_pattern_raises` |
| LLM CLI not found (`list_negative`) | `FileNotFoundError` (propagated) | `TestListNegative.test_claude_not_found_propagates` |
| Malformed LLM JSON response | Graceful: returns `count=0`, **no exception** | `TestListNegative.test_malformed_llm_response` |
| LLM returns unknown node IDs | Graceful: filtered out, **no exception** | `TestListNegative.test_llm_returns_unknown_ids` |

The pattern is clear: **structural errors** (missing nodes, duplicate IDs, invalid state transitions) raise exceptions. **Semantic uncertainty** (LLM returned garbage, drift detected at accept time) is handled gracefully via status fields (`"stale"`, `applied=False`) rather than exceptions.

---

## Topics to Explore

- [file] `reasons/api.py` — The module under test; understanding its function signatures, the `_with_network` context manager, `_fts_search`/`_fts_query` internals, and the proposal state machine will make every assertion in this file click
- [file] `reasons/storage.py` — Tests in `TestPropagate` and `TestLifecycleTimestamps` poke directly at `Storage.load()`/`Storage.save()` to inject artificial state; understanding the on-disk schema and node representation is essential
- [function] `reasons/api.py:accept_proposal` — The drift-detection logic (target already OUT, node deleted, ID collision) is the most subtle code path; the four `test_drift_*` tests here are its acceptance criteria
- [general] `fts-progressive-relaxation` — The FTS search pipeline (stop-word filtering → term relaxation → single-char guard → 51-call cap) is a non-trivial algorithm; `TestFtsSearch` exercises every branch
- [file] `reasons/cluster.py` — `TestDeduplicateSemantic` and `TestListClusters` depend on this module's optional ML stack; the `HAS_CLUSTER_DEPS` gate and the Jaccard-vs-semantic fallback are important to understand before running the full suite

## Beliefs

- `retract-cascades-to-transitive-dependents` — Retraacting a node must set `truth_value="OUT"` on all transitive SL dependents, and the `changed` list in the result must contain exactly the affected node IDs
- `accept-drift-returns-stale-not-exception` — If a proposal's target state changed between `propose_*` and `accept_proposal` (already OUT, deleted, ID collision), `accept_proposal` returns `{"applied": false, "status": "stale"}` rather than raising
- `fts-search-bounded-at-51-calls` — The internal `_fts_query` function is invoked at most 51 times per `api.search()` call, regardless of query length, to prevent combinatorial explosion from progressive term relaxation
- `proposal-actions-are-stale-independent` — Proposing a retraction on node X does not stale a pending supersession on X, and vice versa; auto-staling only applies to the same action type on the same target
- `cascade-does-not-stamp-retracted-at` — When a node goes OUT via cascade (its SL antecedent was retracted), its `retracted_at` field remains empty; only explicitly retracted nodes receive a `retracted_at` timestamp
