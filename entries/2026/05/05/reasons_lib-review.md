# File: reasons_lib/review.py

**Date:** 2026-05-05
**Time:** 15:32

## Purpose

`review.py` is a **semantic auditor for derived beliefs**. The TMS engine validates that belief derivations are *structurally* correct (antecedents exist, justification types are valid), but it cannot judge whether the *reasoning* is sound. This module closes that gap by feeding each derived belief and its full justification chain to an LLM, which evaluates three axes: validity, sufficiency, and necessity.

It's the "second opinion" layer — after `derive.py` produces new beliefs via LLM-driven inference, `review.py` checks whether those derivations actually hold up logically.

## Key Components

### `REVIEW_BATCH_SIZE = 20`

Controls how many beliefs are sent per LLM call. Balances context window usage against round-trip overhead.

### `REVIEW_PROMPT`

A structured prompt template that instructs the LLM to act as a TMS auditor. Defines three evaluation axes (valid, sufficient, necessary) and specifies the exact JSON output schema. Uses `{beliefs}` as the single interpolation point. The `{{` / `}}` escaping prevents `.format()` from choking on the JSON example.

### `format_belief_for_review(node_id, nodes) → str`

Renders a single derived belief into a human-readable block for the prompt. Includes:
- The claim text
- Each justification's antecedents (with their texts resolved from `nodes`)
- Each justification's outlist entries (also resolved)
- Justification labels

Handles multi-justification beliefs by numbering them (`Justification 1/3`, etc.). Returns empty string if the node isn't found — a silent no-op rather than an error.

### `parse_review_response(response) → list[dict]`

Extracts a JSON array from LLM output that may contain surrounding prose. Uses `json.JSONDecoder.raw_decode` scanning from each `[` character — this is a robust pattern for pulling structured data from LLM responses that don't always return clean JSON. Each result is normalized to a consistent schema with defaults (`valid=True`, `sufficient=True`, `necessary=True`), so missing fields fail safe toward "no problem detected."

### `review_beliefs(nodes, belief_ids=None, ...) → list[dict]`

The main entry point. Orchestrates the full review pipeline:
1. Selects which beliefs to review (all derived IN beliefs, or a specified subset)
2. Batches them into groups of `batch_size`
3. Formats and sends each batch to the LLM
4. Collects and returns all results

## Patterns

**Batch-and-collect**: Beliefs are processed in fixed-size batches to stay within LLM context limits, with results accumulated across batches. This is the same pattern you'd see in any bulk-LLM-processing pipeline.

**Fail-safe defaults**: `parse_review_response` defaults all flags to `True` (no problems). A partially-parsed result won't generate false alarms.

**Robust JSON extraction**: Rather than requiring the LLM to return *only* JSON, the parser scans for the first valid JSON array. This handles common LLM behaviors like prefixing with "Here are the results:" or appending explanations.

**Soft failure on batch errors**: A failed batch prints a warning to stderr and continues with the next batch, rather than aborting the entire review. The caller gets partial results.

## Dependencies

**Imports**: Only `json`, `sys`, and `invoke_model` from the sibling `llm` module. Extremely lean — no storage, no network, no CLI concerns.

**Imported by**: `reasons_lib/api.py` (the public API surface) and `tests/test_review.py`. The `.venv` imports in the "Imported By" list are false positives from unrelated packages that happen to have modules named `review`.

## Flow

```
review_beliefs(nodes)
  │
  ├─ Filter: select derived IN beliefs with ≥1 justification
  │
  ├─ For each batch of 20:
  │   ├─ format_belief_for_review() for each belief
  │   │   └─ Resolves antecedent/outlist node texts from nodes dict
  │   ├─ Interpolate into REVIEW_PROMPT
  │   ├─ invoke_model() → raw LLM response
  │   └─ parse_review_response() → list of result dicts
  │
  └─ Return: flat list of all review results
```

The `nodes` dict is the in-memory network snapshot (from `export_network()`). The module never touches the database — it operates entirely on the exported dict representation.

## Invariants

- Only beliefs with at least one justification are reviewed — premises are excluded by the filter in `review_beliefs`.
- When `belief_ids` is `None`, only IN beliefs are reviewed. When explicit IDs are passed, truth value is not checked (but justification existence still is).
- Results are returned in batch order, not necessarily matching input order within a batch (depends on LLM compliance).
- Each result dict always has all six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`), guaranteed by normalization in `parse_review_response`.

## Error Handling

**LLM call failures**: Caught by a bare `except Exception` in the batch loop. Prints a warning to stderr, skips the batch, continues. The caller receives no results for that batch and has no way to distinguish "skipped due to error" from "no problems found."

**Malformed LLM output**: `parse_review_response` returns `[]` if no valid JSON array is found. Individual items within the array that lack an `id` key are silently dropped. This means a garbled response produces an empty result rather than an exception.

**Missing nodes**: `format_belief_for_review` renders `(not found in network)` for antecedents/outlist entries that don't exist in the `nodes` dict, so the LLM still sees the reference even if the text is missing.

## Topics to Explore

- [function] `reasons_lib/llm.py:invoke_model` — The LLM dispatch function that this module delegates to; understanding its model routing and timeout behavior is essential
- [function] `reasons_lib/derive.py` — The derivation engine that *produces* the beliefs this module reviews; understanding the derivation process explains what kinds of errors review catches
- [file] `reasons_lib/api.py` — The public API that wires review into the CLI; shows how review results are consumed and acted upon
- [file] `tests/test_review.py` — Test cases reveal edge cases: what happens with empty networks, missing antecedents, malformed LLM output
- [general] `review-result-consumption` — How downstream code (api.py, CLI) acts on review verdicts — does a `valid=false` result trigger retraction, or is it advisory only?

## Beliefs

- `review-only-evaluates-derived-beliefs` — `review_beliefs` filters out premises (nodes without justifications); only derived beliefs with at least one justification are sent for LLM review
- `review-parse-defaults-fail-safe` — `parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True`, so a missing field never triggers a false alarm
- `review-batch-failure-is-silent-skip` — When an LLM call fails for a batch, the error is logged to stderr but the batch is skipped with no indication in the returned results
- `review-has-no-storage-dependency` — The review module operates entirely on an in-memory `nodes` dict and never reads from or writes to the database directly
- `review-result-schema-is-normalized` — Every result dict returned by `parse_review_response` is guaranteed to have exactly six keys (`id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`) regardless of what the LLM returned

