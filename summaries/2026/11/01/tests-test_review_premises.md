---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_review_premises.py

# `tests/test_review_premises.py`

## Purpose

This file is the test suite for the **premise review** subsystem. The subsystem's job is to take *premise nodes* (observations/claims in the knowledge graph that serve as foundational evidence) and use an LLM to verify them against their cited source documents. For each premise, the LLM decides: is the claim **accurate** (faithful to the source)? Is it **well-scoped** (not overgeneralized)? And if not, what **error_type** was detected (`misread_source`, `fabricated`, etc.)?

The tests cover four layers of that pipeline:

1. **Prompt formatting** (`format_premise_for_review`) — building the structured input the LLM sees.
2. **Response parsing** (`parse_premise_review_response`) — extracting a JSON verdict from the LLM's (often prose-wrapped) output.
3. **Batch orchestration** (`review_premises` in `reasons.review_premises`) — the loop that chunks premises, calls the LLM via subprocess, and collects results.
4. **API integration** (`api.review_premises`) — the higher-level entry point that filters premises from the DB, skips ones without sources, optionally stores review metadata, and returns summary counts.

## Key Components

### `TestFormatPremiseForReview`

Tests `format_premise_for_review(node_id, nodes, source_text)` → `str`.

- **Contract**: Returns a Markdown-formatted block containing the node ID as a `###` heading, the premise text, the source reference (if present), and the source content. Returns `""` if the node ID doesn't exist in the `nodes` dict.
- **Edge case verified**: A node without a `"source"` key should omit the `"Source reference:"` line entirely (line 33–36).

### `TestParsePremiseReviewResponse`

Tests `parse_premise_review_response(raw_llm_output)` → `list[dict]`.

- **Contract**: Accepts a raw string from the LLM, extracts the JSON array (even if wrapped in prose), and returns a list of verdict dicts.
- **Defaults for missing fields** (lines 56–60): If the LLM omits fields, the parser fills in: `accurate=True`, `well_scoped=True`, `error_type=None`, `comment=""`. This is a critical safety default — a vague LLM response shouldn't flag a premise as inaccurate.
- **Malformed input** (line 53): Non-JSON garbage returns `[]` rather than raising.

### `TestReviewPremisesBatchLoop`

Tests the core orchestration function `review_premises(nodes, ids, source_contents, model=..., on_batch=...)`.

- **LLM invocation**: The tests mock `shutil.which` and `subprocess.run` in the `reasons.llm` module. The LLM is invoked as a **subprocess call to the `claude` CLI binary** (`/usr/bin/claude`), not via an HTTP SDK. The return object is a duck-typed `type("R", (), {"returncode": 0, "stdout": ..., "stderr": ""})()`.
- **`on_batch` callback** (lines 95–107): An optional callable that receives each batch's results. Tests confirm it fires once per batch with the batch length.
- **Failure isolation** (lines 109–118): If `subprocess.run` raises, the function returns `[]` — a failed batch is swallowed, not propagated.
- **Empty input** (line 93): `review_premises({}, [], {})` short-circuits to `[]`.

### `TestApiReviewPremises`

Tests `api.review_premises(...)` — the DB-backed entry point.

- **Fixture `db_with_premises`** (lines 122–136): Sets up a temp SQLite DB with two premise nodes (`obs-1`, `obs-2`, both with `source=str(source_file)`) and one derived node (`derived-1`, linked via `sl="obs-1"`). The source file contains `"The system uses PostgreSQL for storage."`.
- **Premise filtering** (lines 138–150): `derived-1` must NOT appear in reviewed IDs. `total_premises` reflects only nodes without `sl`.
- **Source-skip logic** (lines 152–157): A node with no `source` field is counted in `skipped_no_source` and excluded from review.
- **`belief_ids` filter** (lines 159–170): Passing a specific list restricts review to those nodes only.
- **`sample` parameter** (lines 172–183): Limits the number of premises reviewed (useful for cost control).
- **Metadata persistence** (lines 185–204): After a non-dry-run review, each node's `metadata` dict gains:
  - `premise_review_result`: the `error_type` value, or `"pass"` if `accurate=True`.
  - `last_premise_reviewed`: a timestamp (asserted via `"in"` check).
- **Dry-run isolation** (lines 206–220): With `dry_run=True`, no metadata is written. The test asserts `"premise_review_result" not in metadata`.
- **Summary counts** (lines 222–239): The return dict includes `inaccurate`, `overgeneralized` (i.e., `well_scoped=False`), and `reviewed` totals.

### `TestCliDispatch`

Two lightweight smoke tests confirming the CLI command `cmd_review_premises` exists and is callable in `reasons.cli`.

## Patterns

| Pattern | Where | Detail |
|---|---|---|
| **Duck-typed mock for subprocess** | All `TestReviewPremisesBatchLoop` and `TestApiReviewPremises` tests | `type("R", (), {"returncode":0, "stdout":..., "stderr":""})()` avoids importing a real `subprocess.CompletedProcess`. Clean, no extra deps. |
| **`patch` at the call site** | Every LLM-dependent test | Patches `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` specifically, not globally. This means the `reasons.llm` module is the single chokepoint for LLM I/O. |
| **Fixture-scoped temp DB** | `db_with_premises` fixture | Uses `tmp_path` (pytest built-in) so each test gets an isolated SQLite file. No cross-test pollution. |
| **Layered testing** | Four test classes | Each class maps to one function in the pipeline: format → parse → batch-loop → API. A bug can be localized to a single layer. |
| **Default-fill on parse** | `test_defaults_for_missing_fields` | The parser is defensive: missing LLM fields get safe defaults rather than raising `KeyError`. This is a contract, not an accident. |
| **`dry_run` as a first-class parameter** | `TestApiReviewPremises` | The API separates *computing* the review from *persisting* it. Tests verify both paths. |

## Dependencies

### This file imports
- `json` — for constructing mock LLM responses.
- `unittest.mock.patch` — for isolating LLM subprocess calls.
- `pytest` — test runner, `tmp_path` fixture.
- `reasons.api` — the high-level API under test (init_db, add_node, review_premises, export_network).
- `reasons.review_premises` — the three functions under test at the module level.
- `reasons.cli` — for the dispatch smoke tests.

### What depends on this file
Nothing imports test files. This is a leaf. However, the **production code it exercises** (`reasons/review_premises.py`, `reasons/api.py`, `reasons/cli.py`, `reasons/llm.py`) is depended upon by:
- `reasons/repair_premises.py` — likely consumes review results to trigger repairs.
- `reasons/cli.py` — dispatches the `review-premises` command.
- `reasons_lib/` — a parallel copy of the library (the repo maintains two import paths: `reasons.*` and `reasons_lib.*`).

## Flow

Here's the end-to-end execution path the tests validate:

```
User calls:  api.review_premises(db_path=..., belief_ids=[...], sample=N, dry_run=False)
                │
                ├── 1. Load all nodes from DB
                ├── 2. Filter: keep only premises (no `sl` field)
                ├── 3. Filter: keep only those with a `source` field
                ├── 4. Apply `belief_ids` filter if given
                ├── 5. Apply `sample` limit if given
                ├── 6. Read source file contents from disk
                │
                ▼
             review_premises(nodes, ids, source_contents, model, on_batch)
                │
                ├── 7. Partition `ids` into batches
                ├── 8. For each batch:
                │       a. format_premise_for_review() for each node → Markdown prompt
                │       b. Invoke LLM via `reasons.llm` → subprocess `claude` CLI
                │       c. parse_premise_review_response(stdout) → list[dict]
                │       d. If `on_batch` callback: call it with batch results
                │       e. If subprocess raises: skip batch, continue
                │
                ▼
             Back in api.review_premises:
                │
                ├── 9. If NOT dry_run:
                │       a. For each result: set node.metadata["premise_review_result"]
                │          = error_type or "pass"
                │       b. Set node.metadata["last_premise_reviewed"] = timestamp
                │       c. Persist to DB
                │
                ├── 10. Compute summary counts:
                │       reviewed, inaccurate, overgeneralized, skipped_no_source
                │
                └── 11. Return {results: [...], reviewed, inaccurate,
                          overgeneralized, skipped_no_source, total_premises}
```

## Invariants

1. **Only premises are reviewed**: Nodes with a `sl` (supported-by) link are *derived* and never entered into the review pipeline. `test_filters_premises_only` enforces this.
2. **Premises without a source are never sent to the LLM**: They are counted in `skipped_no_source` and excluded. This is a precondition — the LLM needs source text to verify against.
3. **Safe defaults on parse**: A missing `accurate` field defaults to `True` (innocent until proven guilty). A missing `error_type` defaults to `None`. This prevents a lazy LLM response from triggering false-positive repairs.
4. **Dry-run is read-only**: When `dry_run=True`, no metadata is written to the DB. `test_dry_run_no_metadata` asserts the key is absent from the exported network.
5. **Batch failures are non-fatal**: A single batch's subprocess exception does not abort the remaining batches. The function returns whatever results it collected (possibly `[]`).
6. **The LLM is invoked as a subprocess**, not an in-process SDK call. The `claude` binary path is resolved via `shutil.which` at call time, making it configurable via `PATH`.

## Error Handling

| Scenario | Behavior | Test |
|---|---|---|
| LLM returns non-JSON | `parse_premise_review_response` returns `[]` — no exception | `test_malformed_json` |
| LLM returns JSON wrapped in prose | Parser extracts the JSON array from surrounding text | `test_json_with_prose` |
| LLM omits fields in a result dict | Defaults are filled: `accurate=True`, `well_scoped=True`, `error_type=None`, `comment=""` | `test_defaults_for_missing_fields` |
| `subprocess.run` raises (LLM binary missing, crash, timeout) | The batch is skipped; `review_premises` returns `[]` or partial results; no exception propagates to the caller | `test_failed_batch_continues` |
| Node ID not in the `nodes` dict | `format_premise_for_review` returns `""` (empty string) | `test_missing_node` |
| Empty `ids` list | `review_premises` returns `[]` immediately | `test_empty_ids_returns_empty` |
| Premise has no `source` field | Excluded from LLM call, counted in `skipped_no_source` | `test_skips_premises_without_source` |

The overarching strategy: **the review pipeline is best-effort and never raises to the caller**. Every failure mode degrades to "this premise wasn't reviewed" rather than crashing the entire batch or the CLI command.

## Topics to Explore

- [file] `reasons/review_premises.py` — The production implementation of `format_premise_for_review`, `parse_premise_review_response`, and the batch loop. Reading the actual prompt template and batch-size logic is essential for understanding what the LLM is asked to do.
- [file] `reasons/repair_premises.py` — The downstream consumer of review results. Understanding how a `misread_source` or `fabricated` verdict triggers a repair closes the loop from detection to remediation.
- [file] `reasons/llm.py` — The subprocess-based LLM invocation layer. This is where the `claude` CLI is called, where `shutil.which` resolves the binary, and where stdout is captured. It's the single I/O chokepoint that all tests mock.
- [function] `reasons/api.py:review_premises` — The API-layer glue that queries the DB, filters premises, reads source files from disk, and decides whether to persist metadata. The tests exercise its contract but the implementation details (batch sizing, file I/O, metadata schema) live here.
- [general] `reasons-vs-reasons_lib-duplication` — The repo contains two parallel packages (`reasons/` and `reasons_lib/`) with identical module names. Understanding which is the source of truth, which is generated, and how CI treats them will prevent editing the wrong copy.

## Beliefs

- `review-pipeline-is-subprocess-based` — All LLM calls in the review pipeline go through a `subprocess.run` invocation of the `claude` CLI binary (resolved via `shutil.which`), not an HTTP SDK or in-process inference; `reasons.llm` is the sole I/O chokepoint.
- `parse-fills-safe-defaults` — `parse_premise_review_response` defaults missing fields to `accurate=True`, `well_scoped=True`, `error_type=None`, `comment=""`, ensuring a sparse LLM response never flags a premise as inaccurate by omission.
- `dry-run-is-read-only` — When `api.review_premises` is called with `dry_run=True`, no `premise_review_result` or `last_premise_reviewed` key is written to any node's metadata in the database.
- `batch-failure-is-isolated` — A subprocess exception in one batch of `review_premises` does not propagate; the function returns a partial result list (or `[]`) and does not raise, so a single LLM failure cannot abort a multi-batch review.
- `premises-exclude-derived-nodes` — The API layer filters out any node that has a non-null `sl` (supported-by) field before passing to the review pipeline; derived nodes are never reviewed as premises.
