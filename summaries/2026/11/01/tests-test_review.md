---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_review.py

# `tests/test_review.py`

## Purpose

This file is the test suite for the **derived belief review** feature. The `reasons.review` module takes a set of derived (i.e., non-premise) beliefs that are currently marked `IN` (in-the-logic), sends them to an LLM for validation, and gets back a structured verdict on three axes:

- **Valid** — does the conclusion actually follow from the antecedents?
- **Sufficient** — do the antecedents provide *enough* support?
- **Necessary** — are *all* antecedents actually needed (or are some redundant)?

The test file's responsibility is to verify every layer of that pipeline: the text formatter that builds the LLM prompt, the JSON parser that handles LLM output, the batched orchestration in `review_beliefs()`, the API-level filtering and metadata persistence, the CLI command (`review-beliefs`), and the downstream effects (retraction, findings files, JSON reports, node metadata, and list-query filters).

## Key Components

| Component | Contract |
|---|---|
| `run_cli(*args, db_path)` | Simulates a full CLI invocation by patching `sys.argv`, `sys.stdout`, `sys.stderr`. Returns `(stdout_str, stderr_str, exit_code)`. Catches `SystemExit` to extract the exit code. |
| `_make_nodes()` | Returns a hand-crafted dict of 6 nodes (3 premises, 2 derived, 1 gated-OUT) used as in-memory fixture data for unit tests that don't need a real DB. |
| `TestFormatBeliefForReview` | Verifies `format_belief_for_review(node_id, nodes)` produces a markdown-formatted string with the belief text, its antecedents (resolved by ID), outlist, and label. Handles missing antecedents, missing nodes (returns `""`), multiple justifications (numbered), and single justification (unnumbered). |
| `TestParseReviewResponse` | Verifies `parse_review_response(raw_str) -> list[dict]`. Accepts JSON embedded in prose, skips items without `"id"`, rejects non-list JSON, fills missing fields with defaults (`valid=True`, `sufficient=True`, `necessary=True`, `unnecessary_antecedents=[]`, `comment=""`, `defeat_reason_type=""`). |
| `TestReviewBeliefs` | Verifies the core `review_beliefs(nodes, belief_ids, batch_size, timeout, model)` function: it only reviews derived `IN` beliefs, respects `batch_size` (counting `subprocess.run` calls), passes `timeout` through, and returns `[]` on LLM errors. |
| `TestReviewBeliefsApi` | Integration-level tests using a real SQLite DB via `api.add_node(...)`. Verifies filtering by `min_depth`, `sample`, `depends_on`, `namespace`, `visible_to`, and that summary counts (`reviewed`, `invalid`, `insufficient`, `unnecessary`) are computed correctly. |
| `TestCmdReviewBeliefs` | End-to-end CLI tests. Verifies `--auto-retract` actually flips `truth_value` to `OUT`, `--dry-run` prevents retraction, `-o` writes a markdown findings file, `--report-dir` writes a JSON report with `status: "complete"`, `--no-report` suppresses report output, and the `on_batch` callback fires per batch. |
| `TestReviewBeliefsMetadata` | Verifies that after a review, `api.show_node(...)` returns `reviewed_at` timestamp and `metadata["review_result"]` set to `"pass"`, `"invalid"`, or `"insufficient"`. Also confirms `dry_run=True` skips metadata writes. |
| `TestListNodesReviewFilters` | Verifies `api.list_nodes(never_reviewed=True)`, `not_reviewed_since=30`, `by_impact=True` (sorts by `dependent_count` desc), and that every listed node carries `last_reviewed` and `review_result` fields. |

## Patterns

- **LLM mocking via `subprocess.run`**: Every test that exercises the LLM call patches `reasons.llm.shutil.which` (to simulate the `claude` CLI being installed) and `reasons.llm.subprocess.run` (to return a canned JSON response). The mock result is a lightweight `type("R", (), {...})` instance — no `subprocess.CompletedProcess` needed. This keeps tests hermetic and fast.

- **Two-tier testing**: Pure-unit tests (`TestFormatBeliefForReview`, `TestParseReviewResponse`) use in-memory dicts from `_make_nodes()`. Integration tests (`TestReviewBeliefsApi`, `TestCmdReviewBeliefs`, `TestReviewBeliefsMetadata`) use a real SQLite file in `tmp_path` with `api.add_node(...)`. This separates logic bugs from I/O bugs.

- **CLI simulation**: `run_cli()` patches `sys.argv`/`sys.stdout`/`sys.stderr` and calls `reasons.cli.main()`. This lets you test the full CLI path (arg parsing → API call → output formatting → DB mutation) without spawning a subprocess.

- **Fixture-per-class DB setup**: Each integration test class defines its own `db_path` fixture, building exactly the nodes it needs. This avoids cross-test contamination.

- **Mock response factory**: `TestCmdReviewBeliefs._mock_review()` wraps the repeated `type("R", ...)` construction so individual test methods just pass the JSON payload.

## Dependencies

**This file imports:**

- `reasons.review` → `format_belief_for_review`, `parse_review_response`, `review_beliefs`, `REVIEW_BATCH_SIZE`
- `reasons.api` → `add_node`, `show_node`, `list_nodes`, `review_beliefs` (the DB-backed API)
- `reasons.cli` → `main` (CLI entry point)
- `reasons.llm` → (indirectly, via patches) `shutil.which`, `subprocess.run`
- `pytest`, `unittest.mock`, `json`, `sys`, `io.StringIO`, `os`

**What depends on this file:** Nothing at runtime — it's a test file. However, it *pins the behavioral contract* of `reasons.review`, `reasons.api.review_beliefs`, and the `review-beliefs` CLI command. Changing any of those without updating these tests will break the suite.

## Flow

The production code path this file exercises:

```
CLI: reasons review-beliefs --auto-retract
  │
  ├─► api.review_beliefs(db_path, auto_retract=True, ...)
  │     │
  │     ├─► Query DB for derived IN beliefs (apply filters: min_depth,
  │     │   sample, depends_on, namespace, visible_to, belief_ids)
  │     │
  │     ├─► For each batch (REVIEW_BATCH_SIZE at a time):
  │     │     │
  │     │     ├─► format_belief_for_review(id, nodes)  → markdown prompt
  │     │     │     (resolve antecedent IDs to "id: text" lines,
  │     │     │      include outlist, label, justification numbering)
  │     │     │
  │     │     ├─► llm.shutil.which("claude") → confirm CLI exists
  │     │     │
  │     │     ├─► subprocess.run([...claude prompt...], timeout=...)
  │     │     │
  │     │     └─► parse_review_response(stdout) → list of verdict dicts
  │     │           (extract JSON array from prose, fill defaults,
  │     │            skip items without "id")
  │     │
  │     ├─► on_batch callback (optional, for partial reports)
  │     │
  │     ├─► Write metadata: reviewed_at, metadata["review_result"]
  │     │   (skipped if dry_run)
  │     │
  │     ├─► If auto_retract and valid=False:
  │     │     └─► api.set_truth_value(id, "OUT")  (skipped if dry_run)
  │     │
  │     └─► Return summary dict: reviewed, total_derived, invalid,
  │         insufficient, unnecessary, results[]
  │
  ├─► Print findings to stdout (INVALID/INSUFFICIENT/UNNECESSARY flags)
  ├─► If -o: write markdown findings file
  └─► If --report-dir and not --no-report: write JSON report
```

## Invariants

1. **Only derived `IN` beliefs are reviewed.** Premises (no justifications) and `OUT` beliefs are excluded. Verified by `test_empty_derived_returns_empty` and `test_filters_to_existing_ids`.

2. **Missing nodes in the network produce a graceful placeholder.** `format_belief_for_review` returns `""` for a nonexistent node ID, and renders `"nonexistent: (not found in network)"` for a dangling antecedent reference. It never raises.

3. **`parse_review_response` is total.** It never raises on malformed input. Bad JSON → `[]`. Non-list JSON (e.g., a bare object) → `[]`. Items without `"id"` are silently skipped. Missing fields get safe defaults (all boolean flags default to `True`, meaning "pass").

4. **Batch size is a hard cap on LLM calls.** With `batch_size=1` and 2 derived beliefs, `subprocess.run` is called exactly 2 times. Verified by `test_batch_size_respected`.

5. **`dry_run` is a global kill-switch for mutations.** It prevents both retraction (DB write) and metadata writes. Verified by `test_dry_run_prevents_retraction` and `test_dry_run_skips_metadata`.

6. **Metadata classification has a strict priority.** If `valid=False`, the result is `"invalid"` regardless of `sufficient`/`necessary`. Otherwise, if `sufficient=False`, it's `"insufficient"`. Otherwise, if `necessary=False`, it's `"unnecessary"`. Otherwise, `"pass"`. Verified by `test_review_result_classification_priority` and `test_insufficient_result_classification`.

7. **`namespace=""` means "local only" (no colon in ID).** Verified by `test_empty_namespace_filters_local`.

## Error Handling

- **LLM process failure** (`subprocess.run` raises `RuntimeError`): `review_beliefs` catches it and returns `[]`. The caller does not crash. Verified by `test_llm_error_continues`.

- **Malformed LLM output** (non-JSON, non-list, missing fields): `parse_review_response` degrades gracefully — returns `[]` or fills defaults. No exception propagates. Verified by `test_malformed_json_returns_empty`, `test_non_list_json_skipped`, `test_missing_fields_get_defaults`.

- **Dangling antecedent references**: `format_belief_for_review` substitutes `"(not found in network)"` rather than raising `KeyError`. Verified by `test_missing_antecedent_graceful`.

- **CLI `SystemExit`**: `run_cli()` catches it to extract the exit code. Tests like `test_no_derived_beliefs_message` confirm the CLI prints a friendly message rather than crashing.

- **Report file I/O**: Tests verify file existence and content but don't test what happens if the directory is unwritable — that's left to the filesystem.

## Topics to Explore

- [file] `reasons/review.py` — The module under test. Read `format_belief_for_review`, `parse_review_response`, and `review_beliefs` to understand the prompt construction, JSON extraction regex, and batch loop that these tests pin down.
- [file] `reasons/api.py` — The DB-backed API layer. Specifically `api.review_beliefs` (which adds filtering, metadata writes, and retraction on top of `reasons.review.review_beliefs`) and `api.list_nodes` (which the `TestListNodesReviewFilters` class exercises).
- [function] `reasons/cli.py:cmd_review_beliefs` — The CLI handler that wires `--auto-retract`, `--dry-run`, `-o`, `--report-dir`, `--no-report`, and `on_batch` together. The tests in `TestCmdReviewBeliefs` define its observable contract.
- [file] `tests/conftest.py` — Shared fixtures and configuration that may affect how `tmp_path`, DB setup, or LLM mocking behave across the suite.
- [general] `review-justifications-vs-review-beliefs` — The repo has both `review.py` and `review_justifications.py` / `review_premises.py`. Understanding how these three review modes differ (and whether they share code) is important before modifying any of them.

## Beliefs

- `review-beliefs-reviews-only-derived-in` — `api.review_beliefs` filters to beliefs that have at least one justification and `truth_value == "IN"`; premises and `OUT` beliefs are never sent to the LLM.
- `parse-review-response-is-total` — `parse_review_response` never raises; malformed JSON, non-list JSON, and items missing `"id"` all result in those entries being silently dropped (returning `[]` or a shorter list).
- `dry-run-prevents-db-mutation` — When `dry_run=True`, neither `truth_value` changes (retraction) nor `reviewed_at`/`metadata["review_result"]` is written; the LLM is still called and results are returned.
- `review-result-classification-has-priority` — The `metadata["review_result"]` value is determined by a strict priority: `valid=False` → `"invalid"` > `sufficient=False` → `"insufficient"` > `necessary=False` → `"unnecessary"` > else `"pass"`.
- `batch-size-bounds-llm-calls` — The number of `subprocess.run` invocations in `review_beliefs` equals `ceil(qualified_belief_count / batch_size)`; the `batch_size` parameter is a hard upper bound per call, not a soft hint.
