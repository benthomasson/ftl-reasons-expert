# File: reasons_lib/review.py

**Date:** 2026-05-08
**Time:** 14:13



# `reasons_lib/review.py` — LLM-Based Belief Audit

## Purpose

This module is the **semantic validation layer** for derived beliefs. The TMS engine guarantees *structural* validity — justifications exist, antecedents are IN, outlist nodes are OUT — but it can't tell you whether the reasoning from antecedents to conclusion actually makes sense. That's what this module does: it feeds derived beliefs and their full justification chains to an LLM and asks "does this conclusion actually follow?"

It evaluates three axes per belief: **validity** (does the conclusion follow?), **sufficiency** (are the antecedents enough?), and **necessity** (are all antecedents load-bearing?).

## Key Components

### Constants

- **`REVIEW_BATCH_SIZE = 20`** — Default number of beliefs per LLM call. Balances token cost against round-trip overhead.

- **`REVIEW_PROMPT`** — The system prompt template. Uses `{beliefs}` as a single format placeholder. Instructs the LLM to return a JSON array with one object per belief containing `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, and `comment`. Notably, it tells the LLM that a belief with multiple justifications is valid if *any* justification is sound — matching Doyle's TMS semantics where alternative support paths are disjunctive.

### Functions

**`format_belief_for_review(node_id, nodes)`** — Serializes a single belief into a human-readable text block for the LLM. Renders the claim text, then each justification with its antecedent texts (resolved by looking up node IDs in `nodes`), outlist entries, and label. Returns empty string if the node doesn't exist. This is the "prepare" step — it turns graph data into prose the LLM can reason about.

**`parse_review_response(response)`** — Extracts a JSON array from the LLM's response text. Uses `json.JSONDecoder.raw_decode` scanning for `[` characters to tolerate preamble prose and trailing text — a pragmatic choice since LLMs often wrap JSON in markdown fences or add commentary. Validates each item is a dict with an `id` key, and normalizes missing fields to safe defaults (`valid=True`, `sufficient=True`, `necessary=True`, empty lists/strings). Returns `[]` on total parse failure.

**`review_beliefs(nodes, belief_ids, model, timeout, batch_size, on_batch)`** — The orchestrator. Filters to derived IN beliefs (those with justifications), batches them, calls the LLM per batch, and accumulates results. The `on_batch` callback enables streaming/incremental persistence — callers can save partial results after each batch completes. Failed batches emit a warning to stderr and are skipped rather than aborting the whole review.

## Patterns

- **Batch-and-accumulate**: Beliefs are processed in fixed-size batches to stay within token limits. Results accumulate in `all_results` and are also passed to `on_batch` after each batch.

- **Lenient JSON parsing**: `parse_review_response` doesn't assume the LLM returns clean JSON. It scans for the first valid JSON array in the response, which is a common pattern when working with LLM outputs that may include markdown fences or explanatory text.

- **Defensive defaults**: Missing fields in the LLM's response default to "everything is fine" (`valid=True`, etc.). This means a parsing glitch won't generate false-positive warnings — it fails safe toward acceptance.

- **Progress to stderr**: Batch progress and warnings go to `sys.stderr`, keeping stdout clean for structured output.

## Dependencies

**Imports:**
- `json`, `sys` — stdlib
- `.llm.invoke_model` — The single LLM abstraction. This module doesn't know or care which model backend is used.

**Imported by:**
- `reasons_lib/api.py` — The public API surface exposes review functionality
- `tests/test_review.py` — Unit tests

(The `.venv` imports in the "Imported By" list are false positives from name collisions — unrelated packages that happen to have a `review` symbol.)

## Flow

1. **Filter**: `review_beliefs` selects which beliefs to review — either the caller-specified IDs or all derived IN beliefs. Non-derived beliefs (premises) and beliefs not in the network are excluded.
2. **Batch**: The filtered list is chunked into groups of `batch_size`.
3. **Format**: Each batch is rendered via `format_belief_for_review`, which resolves antecedent IDs to their text.
4. **Prompt**: The formatted text is injected into `REVIEW_PROMPT` and sent to `invoke_model`.
5. **Parse**: `parse_review_response` extracts structured results from the LLM response.
6. **Accumulate**: Results extend `all_results`; `on_batch` is called if provided.
7. **Return**: The full list of review result dicts.

## Invariants

- Only beliefs with at least one justification are reviewed — premises (no justifications) are always skipped.
- When `belief_ids` is None, only IN beliefs are reviewed. When specific IDs are provided, truth value is not filtered (but justifications are still required).
- Each result dict always has exactly the keys `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment` — `parse_review_response` enforces this shape.
- Batch ordering is preserved — results appear in the same order as input belief IDs (modulo any batches that fail entirely).

## Error Handling

- **LLM failures**: Caught by a bare `except Exception` in the batch loop. The batch is skipped with a warning to stderr. Other batches still proceed. This means a partial review is returned rather than no review.
- **Parse failures**: If `parse_review_response` can't find valid JSON, it returns `[]` — the batch contributes nothing. No exception is raised.
- **Missing nodes**: `format_belief_for_review` returns `""` for missing nodes, and marks missing antecedents with `(not found in network)` rather than crashing. This tolerates stale belief IDs that reference deleted nodes.

## Topics to Explore

- [function] `reasons_lib/llm.py:invoke_model` — The LLM abstraction this module delegates to; understanding its model routing and timeout behavior explains the `model` and `timeout` params here
- [file] `reasons_lib/derive.py` — The "write" side of this module's "read" — `derive` creates the derived beliefs that `review` audits
- [file] `tests/test_review.py` — Shows how the format/parse/review functions are tested in isolation, including edge cases for malformed LLM responses
- [function] `reasons_lib/api.py:review_beliefs` — The public API wrapper that loads the network, calls this module, and persists results to the review JSON files in `reviews/`
- [general] `on-batch-persistence` — How `on_batch` is used by callers to write incremental `review-beliefs-*.json` files, enabling crash recovery during long review runs

## Beliefs

- `review-skips-premises` — `review_beliefs` only processes beliefs that have at least one justification; premise nodes (no justifications) are never sent to the LLM
- `review-parse-defaults-safe` — When the LLM response is missing fields, `parse_review_response` defaults to `valid=True`, `sufficient=True`, `necessary=True` — a parse error never generates a false-positive validity warning
- `review-batch-failure-non-fatal` — A failed LLM batch emits a stderr warning and is skipped; remaining batches still execute and their results are returned
- `review-multi-justification-disjunctive` — The review prompt instructs the LLM that a belief is valid if ANY of its justifications is sound, matching Doyle's TMS disjunctive support semantics
- `review-result-schema-normalized` — Every result dict from `parse_review_response` has exactly six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`) regardless of what the LLM returned

