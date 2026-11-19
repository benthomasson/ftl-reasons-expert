---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/review_justifications.py

# `reasons/review_justifications.py` — LLM-assisted review of ALL vs ANY justification modes

## Purpose

This module is a **read-only auditing tool**. It scans derived beliefs in a Truth Maintenance System (TMS) whose justifications use the multi-antecedent **SL** (justification) type and asks an LLM to classify each justification as:

- **ALL** — conjunctive; every antecedent is required. Retracting any one retracts the conclusion.
- **ANY** — disjunctive; each antecedent independently supports the conclusion.
- **MIXED** — a required core plus optional independent support.

It **reports** candidate misclassifications; it never mutates the database. It's a discovery/repair-adjacent helper, consistent with the other `review_*` modules in the repo (`review.py`, `review_premises.py`).

## Key Components

### Constants

- **`REVIEW_BATCH_SIZE = 15`** — default number of beliefs bundled into one LLM prompt. Chunking bounds prompt size and cost.
- **`REVIEW_JUSTIFICATIONS_PROMPT`** — a large f-string template (uses `{{ }}` escaping for literal JSON braces). Contains the classification rules, the exact JSON output schema, and a `{beliefs}` placeholder. The prompt is the actual "spec" the LLM must obey; changing classification semantics happens here.

### Functions

- **`parse_justification_review(response) -> list[dict]`**
  Scans the LLM response for the first `[` it can `json.JSONDecoder.raw_decode` successfully. For each parsed dict with an `id`, it normalizes into a canonical dict:
  - `classification` uppercased if string, else default `"ALL"`
  - `required_antecedents` / `independent_antecedents` default to `[]` if not lists
  - `comment` default `""`
  Returns the first non-empty parse result, or `[]`. This is deliberately **lenient** — it tolerates prose before/after JSON and partial schema.

- **`_process_batch(batch, nodes, model, timeout) -> list[dict]`**
  Builds the belief text via `format_belief_for_review(nid, nodes)` joined by blank lines, formats the prompt, calls `invoke_model`, and parses. If parsing yields nothing while a non-empty batch was sent, prints a warning to `stderr` and returns `[]` (does not raise).

- **`review_justifications(nodes, belief_ids=None, model="claude", timeout=300, batch_size=REVIEW_BATCH_SIZE, min_antecedents=2, on_batch=None, parallel=0) -> list[dict]`**
  The public entry point. See *Flow* below.

- **`_has_multi_antecedent_sl(node, min_antecedents) -> bool`**
  Predicate: true if the node has **any** justification with `type == "SL"` and `len(antecedents) >= min_antecedents`.

## Patterns

- **Batching / fan-out-fan-in** — belief IDs are chunked into `batch_size` lists, each batch is one LLM call.
- **Callback for progress** — `on_batch(all_results)` is invoked after every batch, passing the *cumulative* list so callers (CLI, API) can stream results.
- **Optional parallel execution** — when `parallel > 0`, a `ThreadPoolExecutor(max_workers=parallel)` is used with `as_completed`; each future runs `_process_batch` independently. LLM calls are I/O-bound, so threads are appropriate.
- **Tolerant parsing** — `parse_justification_review` is a "best-effort salvage" parser that skips invalid JSON candidates and emits a normalized schema regardless of what the LLM produced.
- **Prompt-as-contract** — classification rules live in the prompt template; the Python code only validates shape (defaults + type coercion), not semantics.

## Dependencies

**Imports**
- `json`, `sys`, `concurrent.futures.{ThreadPoolExecutor, as_completed}` (imported lazily inside the parallel branch).
- `reasons.llm.invoke_model` — the actual LLM dispatch (prompt in, text out).
- `reasons.review.format_belief_for_review` — renders a node into the human-readable block used in the prompt.

**Imported by**
- `reasons/api.py` — likely exposes this via the public API surface.
- `reasons_lib/review_justifications.py` — the parallel library mirror (this repo keeps `reasons/` and `reasons_lib/` in sync).
- `tests/test_review_justifications.py` — unit tests for parsing/batching.

Note there is **no direct DB access** — all state comes via the `nodes` dict (produced by `export_network()` per the docstring).

## Flow

1. **Selection**
   - If `belief_ids` is `None`, iterate every node in `nodes` and keep those with `truth_value == "IN"` **and** `_has_multi_antecedent_sl(node, min_antecedents)`.
   - If `belief_ids` is provided, apply the same filters to just those IDs (unknown IDs are silently dropped).
   - Empty selection → return `[]` immediately (no LLM call, no stderr noise).

2. **Batching** — slice `belief_ids` by `batch_size` into a `batches` list.

3. **Execution**
   - **Serial** (`parallel == 0`): loop over batches, log progress to stderr, call `_process_batch`, extend `all_results`, invoke `on_batch(all_results)`. Exceptions caught per-batch.
   - **Parallel** (`parallel > 0`): submit all batches, then iterate `as_completed`. Results are appended in **completion order** — so the returned `all_results` is not necessarily in input order. Logs completion counts; exceptions caught per-future.

4. **Return** — `all_results`, a flat list of normalized dicts (each event carries `id`, `classification`, `required_antecedents`, `independent_antecedents`, `comment`).

## Invariants

- **Only `IN` beliefs are reviewed.** Both selection branches require `truth_value == "IN"`.
- **Only multi-antecedent SL justifications qualify.** `_has_multi_antecedent_sl` requires `len(antecedents) >= min_antecedents` (default `2`; single-antecedent SL justifications are inherently mode-agnostic and skipped).
- **Results are canonicalized** by `parse_justification_review`: `classification` is always uppercase, antecedent lists are always lists, `comment` is always a string.
- **Order within a batch** is expected to match the presented order (prompt rule: "one object per belief reviewed, in the same order as presented"), but **cross-batch order is not guaranteed in parallel mode**.
- **One parse per response**: `parse_justification_review` returns the *first* parseable array containing at least one dict with `id`. Later JSON arrays in the same response are ignored.

## Error Handling

- **Batch failures are swallowed per-batch**: both the serial and parallel paths wrap `_process_batch`/`future.result()` in `try/except Exception`, printing `"WARN: batch N failed: ..."` to `stderr`. A failure does **not** abort the whole review and does **not** propagate to the caller.
- **Parse failures** are handled by returning `[]` from `parse_justification_review`; `_process_batch` logs `"WARN: batch returned no parseable results"` when a non-empty batch produced nothing.
- **`invoke_model` errors** bubble up out of `_process_batch` and are caught by the batch-level guard — so a single serious error (e.g. timeout, rate-limit) is logged and the run continues.
- **No exceptions are raised by `review_justifications` itself** for the standard operational failures; the function returns whatever succeeded. Callers cannot distinguish "everything failed" from "nothing matched the filter" except by inspecting stderr / result emptiness.

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — How prompts are actually dispatched and what error types can be raised (drives the batch `except Exception` behavior).
- [function] `reasons/review.py:format_belief_for_review` — The exact serialization presented to the LLM; this is the input the classification rules act on.
- [file] `reasons/api.py` — How `review_justifications` is surfaced to external callers (parameters, return shape, streaming via `on_batch`).
- [file] `tests/test_review_justifications.py` — Existing tests pin down parsing tolerance and batching edge cases; useful before changing the prompt or schema.
- [general] `all-vs-any-justification-semantics` — How ANY justifications are represented in the underlying storage versus the ALL-only assumption in this prompt (`REVIEW_JUSTIFICATIONS_PROMPT` says "currently all use ALL"); this assumption may become wrong as ANY storage is introduced.

## Beliefs

- `review-justifications-filters-to-in-truth-value` — Both the default and explicit `belief_ids` branches require `node.get("truth_value") == "IN"` before including a node.
- `review-justifications-requires-multi-antecedent-sl` — A node is eligible only if it has at least one justification with `type == "SL"` and `len(antecedents) >= min_antecedents` (default 2).
- `parse-justification-review-returns-first-valid-array` — `parse_justification_review` scans for the first bracket with a successfully decoded JSON list containing an item with `id`, and ignores subsequent arrays.
- `batch-failures-do-not-abort-review` — A `_process_batch` exception inside either the serial loop or a parallel `future.result()` is caught, logged to stderr, and does not stop remaining batches.
- `on-batch-callback-receives-cumulative-results` — `on_batch` is invoked after each batch with the full accumulated `all_results`, not just the newest batch's results.
- `parallel-mode-loses-batch-order` — When `parallel > 0`, results are appended in `as_completed` order, so the returned list is not necessarily in the original `belief_ids` order.
