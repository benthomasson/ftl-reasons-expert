---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_repair_premises.py

# `tests/test_repair_premises.py`

## Purpose

This file is the test suite for the **premise repair pipeline** — the mechanism that takes premises (observations/claims) flagged as inaccurate by a review pass and asks an LLM to either **rewrite** them (correct the text to match the source) or **retract** them (remove them because they have no source support). It covers three layers:

1. **Prompt construction** (`format_premise_for_repair`) — building the LLM input
2. **Response parsing** (`parse_repair_response`) — extracting structured actions from LLM output
3. **End-to-end API** (`api.repair_premises`) — orchestrating the full cycle against a real SQLite DB

The responsibility it owns: ensuring that the repair loop correctly identifies which nodes to fix, produces valid LLM interactions, persists the right mutations (supersede + successor for rewrites, mark-OUT for retracts), stamps metadata, and degrades gracefully on LLM failure.

## Key Components

| Component | Contract |
|-----------|----------|
| `format_premise_for_repair(node_id, nodes, source_content, review)` | Returns a markdown-formatted prompt string containing the node text, error type, optional review comment, and the full source text. Returns `""` if the node ID isn't in `nodes`. |
| `parse_repair_response(response_text)` | Accepts raw LLM stdout (may contain surrounding prose). Returns a list of dicts, each with keys `id`, `action` (`"rewrite"` or `"retract"`), `corrected_text` (str or None), `rationale` (str). Returns `[]` on unparseable input. Defaults: action=`"retract"`, corrected_text=None, rationale=`""`. |
| `repair_premises(nodes, premise_ids, source_contents, review_results, parallel=1, on_result=None)` | Core loop. For each premise ID, calls the LLM and collects a result dict. Supports parallel execution via `parallel` kwarg. Calls `on_result` callback per batch. On LLM exception, returns a single `{"action": "error", ...}` entry. |
| `api.repair_premises(review_file=None, belief_ids=None, dry_run=False, db_path=None)` | High-level orchestrator. Loads the review file, filters inaccurate premises, calls the repair loop, then **mutates the DB**: rewrites create a successor node (old → OUT, new → IN); retracts mark the node OUT. Stamps `metadata.repair_action` as `"rewritten"` or `"retracted"`. Raises `ValueError` if neither `review_file` nor `belief_ids` is provided. |
| `TestCliDispatch.test_repair_premises_registered` | Asserts `cli.cmd_repair_premises` exists and is callable — guards against accidental removal of the CLI entry point. |

## Patterns

- **Subprocess mocking via `patch`**: Every LLM call goes through `reasons.llm` which shells out to a `claude` binary. Tests patch both `shutil.which` (to fake the binary being installed) and `subprocess.run` (to return canned JSON). This isolates tests from any real LLM.

- **`tmp_path`-scoped databases**: Integration-style tests (`TestApiRepairPremises`, `TestRepairActionMetadata`) create a real SQLite DB in a temp directory, seed it with `api.init_db` + `api.add_node`, then assert on the post-repair state via `api.export_network` / `api.show_node`. No shared state between tests.

- **Dry-run vs. commit duality**: Each mutating test has a counterpart with `dry_run=True` that asserts the DB is **unchanged**. This is the standard pattern for "safe preview" commands in this codebase.

- **Callback-based progress**: `on_result` is a per-batch callback (not per-node), tested via `lambda r: callbacks.append(len(r))`. This decouples progress reporting from the repair logic.

- **Side-effect discriminator mocks**: In `test_parallel_repair` and `test_rewrite_updates_node`, the `mock_run` function inspects `kwargs["input"]` (the prompt text) to decide which node is being repaired, returning different JSON. This simulates multiple concurrent LLM calls without a real server.

- **Fixture duplication for isolation**: `db_with_reviewed` is defined independently in both `TestApiRepairPremises` and `TestRepairActionMetadata` rather than hoisted to a shared fixture. This keeps each class self-contained but means the setup code is duplicated — a maintenance smell if the schema changes.

## Dependencies

**Imports (this file → others):**
- `reasons.api` — `init_db`, `add_node`, `repair_premises`, `export_network`, `show_node`
- `reasons.repair_premises` — `format_premise_for_repair`, `parse_repair_response`, `repair_premises`
- `reasons.llm` (indirectly, via patch targets) — `shutil.which`, `subprocess.run`
- `reasons.cli` — `cmd_repair_premises` (existence check only)
- `reasons.review_premises` — `review_premises` (parallel batching test)

**Dependents (what imports this file):**
- The test runner (pytest). No production code imports tests.
- `tests/conftest.py` may provide shared fixtures, though this file appears self-sufficient.

## Flow

```
api.repair_premises(review_file)
  │
  ├─► Load review JSON → filter results where accurate == False
  │     → collect (node_id, error_type, comment) tuples
  │
  ├─► For each node (sequential or parallel batches):
  │     │
  │     ├─► format_premise_for_repair(id, nodes, source, review)
  │     │     → markdown prompt string
  │     │
  │     ├─► reasons.llm.call(prompt)  [subprocess → claude binary]
  │     │     → raw stdout text
  │     │
  │     └─► parse_repair_response(stdout)
  │           → [{"id": "obs-1", "action": "rewrite", "corrected_text": "...", "rationale": "..."}]
  │
  ├─► For each parsed result (if not dry_run):
  │     │
  │     ├─ action == "rewrite":
  │     │     ├─ Mark old node → truth_value = "OUT" (superseded)
  │     │     ├─ Create new node with corrected_text, truth_value = "IN"
  │     │     └─ Set metadata.repair_action = "rewritten" on new node
  │     │
  │     └─ action == "retract":
  │           ├─ Mark node → truth_value = "OUT"
  │           └─ Set metadata.repair_action = "retracted"
  │
  └─► Return summary dict:
        { total_inaccurate, rewritten, retracted, results: [...] }
```

## Invariants

1. **Node existence guard**: `format_premise_for_repair` returns `""` for unknown node IDs — the caller must not crash on missing nodes.
2. **Default action is `retract`**: If the LLM omits the `action` field, the system treats it as a retraction (safe default — never fabricate a rewrite).
3. **`corrected_text` is `None` for retracts**: A retract result must not carry text; the parser enforces this via defaults.
4. **Dry-run is read-only**: When `dry_run=True`, no node is created, no `truth_value` changes, and no `repair_action` metadata is written.
5. **At least one of `review_file` or `belief_ids` must be provided**: Enforced by `ValueError` in `api.repair_premises`.
6. **Rewrite produces a successor, not an in-place edit**: The old node is superseded (OUT); a new node carries the corrected text. This preserves audit history in the belief network.
7. **Empty premise list → empty results**: `repair_premises({}, [], {}, {})` returns `[]` without calling the LLM.

## Error Handling

| Failure mode | Behavior |
|---|---|
| LLM subprocess raises (e.g., binary missing, network down) | `repair_premises` catches the exception and returns `[{"id": ..., "action": "error", ...}]` — the test `test_llm_failure_returns_error` verifies this. No exception propagates to the caller. |
| Malformed JSON in LLM response | `parse_repair_response` returns `[]`. The caller sees zero results for that batch. No crash. |
| Prose around JSON in LLM response | `parse_repair_response` handles it (tested in `test_json_with_prose`) — likely uses a JSON-extraction regex or `json.loads` after stripping. |
| Missing node in `nodes` dict | `format_premise_for_repair` returns `""`; the repair loop presumably skips empty prompts. |
| Neither `review_file` nor `belief_ids` given | `api.repair_premises` raises `ValueError` immediately (tested in `test_no_args_raises`). |

The general strategy: **parse-level failures are swallowed into empty/partial results; invocation-level failures are caught and converted to an `"error"` action dict; argument-level violations raise immediately.**

## Topics to Explore

- [file] `reasons/repair_premises.py` — The production implementation of `format_premise_for_repair`, `parse_repair_response`, and the parallel loop. Understanding its batching logic and how it invokes `reasons.llm` will clarify what the mocks are standing in for.
- [function] `reasons/api.py:repair_premises` — The orchestrator that bridges the review file → repair loop → DB mutations. This is where the supersede/successor pattern and metadata stamping live.
- [file] `reasons/review_premises.py` — The upstream step that *produces* the `review_results` this file consumes. Reading it end-to-end shows the full audit → repair pipeline.
- [general] `supersede-successor-pattern` — How this codebase models "correction" as a new node rather than an in-place edit. Explore `reasons/network.py` and `reasons/verify.py` to see how downstream inferences react to a node going OUT.
- [file] `reasons/llm.py` — The subprocess shim that all LLM calls go through. Understanding its retry/timeout behavior explains why the tests mock at the `subprocess.run` level rather than at a higher API boundary.

## Beliefs

- `repair-rewrite-creates-successor` — A `rewrite` action never mutates the original node's text; it supersedes the old node (OUT) and creates a new node (IN) with the corrected text, preserving the belief-network audit trail.
- `parse-repair-defaults-to-retract` — When the LLM omits the `action` field in its JSON response, `parse_repair_response` defaults to `"retract"` with `corrected_text=None`, making the safe action the default.
- `dry-run-prevents-mutation` — Passing `dry_run=True` to `api.repair_premises` guarantees zero writes to the database: no truth_value changes, no new nodes, no `repair_action` metadata.
- `repair-llm-failure-produces-error-dict` — If the LLM subprocess raises an exception, `repair_premises` returns a result entry with `action="error"` rather than propagating the exception to the caller.
- `test-mocks-subprocess-not-http` — All LLM interactions in this codebase go through `subprocess.run` (shelling out to a `claude` binary), so tests patch `subprocess.run` and `shutil.which` rather than mocking an HTTP client.
