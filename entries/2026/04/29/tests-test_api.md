# File: tests/test_api.py

**Date:** 2026-04-29
**Time:** 17:17

## Purpose

`tests/test_api.py` is the integration test suite for `reasons_lib.api` — the functional Python API that wraps the TMS (Truth Maintenance System) engine. It validates every public function in the API module: database lifecycle, node CRUD, truth maintenance operations (retract/assert with cascades), nogood recording, network export, and LLM-assisted negative-belief classification.

This file owns the contract between external consumers and the TMS. If a behavior is tested here, it's a public guarantee.

## Key Components

Each test class maps 1:1 to an API function:

| Class | API function | What it validates |
|---|---|---|
| `TestInitDb` | `api.init_db()` | DB creation, idempotency guard, force-overwrite |
| `TestAddNode` | `api.add_node()` | Premise vs. SL-derived creation, duplicate rejection |
| `TestRetractNode` | `api.retract_node()` | Single retraction, cascade propagation, missing/already-out edge cases |
| `TestAssertNode` | `api.assert_node()` | Restoration with cascade, already-IN no-op |
| `TestGetStatus` | `api.get_status()` | Empty DB, node counting |
| `TestShowNode` | `api.show_node()` | Full node detail retrieval including `source`, `justifications`, `dependents` |
| `TestExplainNode` | `api.explain_node()` | Trace output for premises and derivation chains |
| `TestAddNogood` | `api.add_nogood()` | Contradiction recording, auto-generated IDs, side-effect changes |
| `TestGetBeliefSet` | `api.get_belief_set()` | Returns only IN node IDs |
| `TestGetLog` | `api.get_log()` | Audit log retrieval, `last=N` truncation |
| `TestExportNetwork` | `api.export_network()` | Full network serialization to dict |
| `TestEndToEnd` | Multiple | Three-node chain: retract root → all cascade OUT → assert root → all restore IN |
| `TestListNodesDepth` | `api.list_nodes()` | `min_depth`/`max_depth` filtering by derivation depth |
| `TestListGated` | `api.list_gated()` | GATE belief inspection — active blockers, satisfied gates, superseded exclusion, multi-gated-per-blocker |
| `TestListNegative` | `api.list_negative()` | LLM-based negative-sentiment classification with keyword pre-filter |

## Patterns

**Isolated DB per test.** The `db_path` fixture creates a fresh SQLite database in `tmp_path` for every test. No shared state, no ordering dependencies, no cleanup needed.

**Return-value-based API.** Every API function returns a dict rather than printing or mutating global state. Tests assert on dict keys (`result["changed"]`, `result["truth_value"]`), making the contract explicit and machine-readable.

**Exception-as-contract.** Error conditions are tested with `pytest.raises` — `FileExistsError` for duplicate DBs, `ValueError` for duplicate nodes, `KeyError` for missing nodes. These are part of the public API contract, not incidental.

**Mock boundary at the LLM.** `TestListNegative` patches `reasons_lib.ask._invoke_claude` to isolate the LLM call. The tests exercise the full pipeline (keyword pre-filter → LLM classification → ID validation → result assembly) without requiring an actual Claude CLI. This is the only class that uses mocking.

**Edge-case coverage for LLM output.** The `TestListNegative` class is notably thorough about handling unreliable LLM responses: malformed JSON, unknown IDs in the response, multiline JSON, and prose-with-brackets-before-JSON. This reflects the real brittleness of LLM-in-the-loop features.

## Dependencies

**Imports:**
- `reasons_lib.api` — the module under test (all public functions)
- `reasons_lib.ask._invoke_claude` — patched in `TestListNegative` (private LLM call helper)
- `pytest` — fixtures, `raises`, parametrization
- `unittest.mock.patch` — LLM isolation

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test follows this pattern:

1. `db_path` fixture creates a temp SQLite DB via `api.init_db()`
2. Test adds nodes with `api.add_node()`, building a small belief network
3. Test performs the operation under test (retract, assert, export, etc.)
4. Test asserts on the returned dict's structure and values

The `TestEndToEnd.test_retract_and_restore_chain` test demonstrates the full TMS lifecycle: build a three-node derivation chain `a → b → c`, verify all IN, retract the root, verify cascade makes all OUT, re-assert the root, verify cascade restores all IN.

For `TestListNegative`, the flow is: add nodes → patch the LLM → call `api.list_negative()` → assert the pipeline correctly pre-filters by keyword, sends the right prompt to the LLM, parses the response, and validates returned IDs against the actual DB.

## Invariants

- **Retraction cascades are total.** Retracting a premise retracts all transitively dependent derived nodes (`test_retract_cascades`, `test_retract_and_restore_chain`).
- **Assertion restores cascades.** Re-asserting a retracted premise restores the full dependency chain (`test_assert_restores`).
- **Idempotent operations return empty `changed`.** Retracting an already-OUT node or asserting an already-IN node returns `changed == []`.
- **Duplicate node IDs are rejected.** `add_node` with an existing ID raises `ValueError`.
- **Superseded nodes are excluded from gated listings.** `test_superseded_excluded` ensures stale conclusions don't pollute the blocker view.
- **`list_negative` never surfaces IDs that don't exist in the DB.** Even if the LLM hallucinates node IDs, they're filtered out (`test_llm_returns_unknown_ids`).
- **Access tags restrict visibility.** `test_visible_to` confirms that `list_negative(visible_to=["public"])` excludes `internal`-tagged nodes from both the result and the LLM prompt.

## Error Handling

The API uses standard Python exceptions as its error protocol:

| Condition | Exception | Test |
|---|---|---|
| `init_db` on existing path | `FileExistsError` | `test_refuses_existing` |
| `add_node` with duplicate ID | `ValueError` | `test_add_duplicate_raises` |
| `retract_node` on nonexistent ID | `KeyError` | `test_retract_missing_raises` |
| `show_node` on nonexistent ID | `KeyError` | `test_show_missing_raises` |
| `claude` CLI not in PATH | `FileNotFoundError` | `test_claude_not_found_propagates` |

Malformed LLM output is handled gracefully — `test_malformed_llm_response` shows that unparseable responses yield `count == 0` rather than raising.

## Topics to Explore

- [file] `reasons_lib/api.py` — The implementation behind every function tested here; understanding the return-dict contracts in detail
- [function] `reasons_lib/api.py:list_negative` — The most complex API function: keyword pre-filter → LLM classification → ID validation pipeline
- [function] `reasons_lib/api.py:list_gated` — GATE belief mechanics including `unless` outlists and `supersede` exclusion
- [file] `tests/test_network.py` — Lower-level TMS engine tests; `TestEndToEnd` explicitly references parity with this file
- [general] `outlist-truth-maintenance` — How `unless` parameters create non-monotonic reasoning (nodes that flip IN when their blocker goes OUT)

## Beliefs

- `api-retract-cascade-is-transitive` — `api.retract_node` propagates OUT to all transitively dependent SL-derived nodes, not just direct children
- `api-list-negative-filters-hallucinated-ids` — `list_negative` discards any node IDs returned by the LLM that don't exist in the database
- `api-list-negative-graceful-on-malformed-llm` — When the LLM returns unparseable output, `list_negative` returns count 0 rather than raising an exception
- `api-superseded-nodes-excluded-from-gated` — `list_gated` omits nodes that have been superseded via `api.supersede()`, even if they still have active blockers
- `api-visible-to-filters-both-result-and-prompt` — `list_negative(visible_to=...)` excludes access-tagged nodes from both the final result and the text sent to the LLM

