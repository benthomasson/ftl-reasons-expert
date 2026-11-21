---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: review_beliefs in reasons/review.py

# `review_beliefs`

## Purpose

`review_beliefs` performs a semantic audit of derived beliefs in a Truth Maintenance System (TMS). Structural validation has already confirmed that antecedents exist, but this function asks an LLM to judge whether each derivation is actually sound on three axes:

- **valid**: does the conclusion follow from the antecedents?
- **sufficient**: are the listed antecedents enough?
- **necessary**: are all antecedents load-bearing?

It exists to catch plausible-sounding but logically weak derivations by sending belief claims plus their justification chains to an LLM and returning structured review results.

## Contract

### Preconditions

- `nodes` is a mapping of `node_id -> node_data`, typically from `export_network()`.
- Each node value is expected to behave like a dict and may contain:
  - `truth_value`
  - `justifications`: a list of justification dicts
  - `text`
- Each justification dict may contain:
  - `antecedents`
  - `outlist`
  - `label`
- `belief_ids` is either `None` or an iterable of node IDs.
- `model` is a model identifier accepted by `invoke_model`.
- `timeout` is a numeric timeout value passed through to `invoke_model`.
- `batch_size` is intended to be a positive integer.
- `on_batch`, if provided, is callable and accepts the cumulative result list.

### Postconditions

- Returns a list of normalized review-result dictionaries.
- If no eligible beliefs remain after filtering, returns `[]` without calling the LLM.
- If some batches fail, returns results from batches that succeeded; failed batches are logged to `stderr` and skipped.
- Only nodes with at least one justification are reviewed.
- When `belief_ids is None`, only nodes with `truth_value == "IN"` and non-empty `justifications` are selected.
- When `belief_ids` is explicitly provided, the function filters for existence in `nodes` and non-empty `justifications`, but does **not** require `truth_value == "IN"`.

### Invariants

- `all_results` is cumulative across batches.
- `on_batch`, when provided, receives the same mutable `all_results` list after each processed batch.
- Batch iteration slices the filtered `belief_ids` in order.
- Duplicate IDs in an explicit `belief_ids` list are preserved and can be reviewed multiple times.
- The function does not mutate `nodes` or the caller’s `belief_ids` list.

### Assumptions not enforced by the type system

- `nodes` values are dict-like and have expected fields.
- `justifications` is a list-like object when non-empty.
- Node IDs are sortable when `belief_ids is None`, because the default path calls `sorted(nodes.items())`.
- `batch_size` is positive; there is no runtime guard.
- LLM results actually correspond to the requested belief IDs; `parse_review_response` only requires an `"id"` key and does not verify membership in the batch.
- Returned booleans and list fields have expected types; `parse_review_response` applies defaults but does not type-check values deeply.

## Parameters

### `nodes`

Mapping of node ID to node data. Used both for selecting beliefs and for formatting antecedent text inside the review prompt.

### `belief_ids`

Optional list of specific belief IDs to review.

- If `None`, selects all nodes where:
  - `truth_value == "IN"`
  - `justifications` is truthy
  - `len(justifications) > 0`
- If provided, only keeps IDs that:
  - exist in `nodes`
  - have non-empty `justifications`

Explicit IDs are not required to be `IN`. This is useful for reviewing arbitrary beliefs, but it means the function can review beliefs that are not currently derivable IN beliefs.

### `model`

Passed directly to `invoke_model`. Default is `"claude"`.

### `timeout`

Passed directly to `invoke_model`. Default is `300` seconds.

### `batch_size`

Number of beliefs per LLM call. Default is `REVIEW_BATCH_SIZE`, currently `20`.

There is no validation that it is positive. If `batch_size == 0`, the function can raise `ZeroDivisionError` while computing `total_batches`. If `batch_size` is negative, `range(0, len(belief_ids), batch_size)` produces no iterations, so the function returns `[]` despite eligible beliefs.

### `on_batch`

Optional callback invoked after each batch’s results are appended to `all_results`.

It receives the cumulative list, not just the current batch’s results. Because the same list object is passed each time, a callback that mutates it affects the final return value and later callback invocations.

If `on_batch` raises, that exception is caught by the per-batch `try/except`, logged as a batch failure, and later batches continue. The current batch’s results are already in `all_results` at that point.

## Return Value

Returns a list of review-result dictionaries produced by `parse_review_response`.

`parse_review_response` normalizes each item to include:

```python
{
    "id": item["id"],
    "valid": item.get("valid", True),
    "sufficient": item.get("sufficient", True),
    "necessary": item.get("necessary", True),
    "unnecessary_antecedents": item.get("unnecessary_antecedents", []),
    "comment": item.get("comment", ""),
    "scope_findings": item.get("scope_findings", []),
    "missing_property": item.get("missing_property", ""),
    "defeat_reason_type": item.get("defeat_reason_type", ""),
}
```

The caller must handle:

- partial results if some batches fail
- empty results if no eligible beliefs exist
- empty results if all batches fail
- empty or partial results if the LLM returns malformed JSON
- results whose IDs may not correspond to the requested batch, since that is not verified
- possible non-boolean values if the LLM returns unexpected types, because parsing only supplies defaults

## Algorithm

1. **Select belief IDs**
   - If `belief_ids is None`:
     - Sort all `(node_id, node_data)` pairs.
     - Keep nodes with `truth_value == "IN"` and non-empty `justifications`.
   - Otherwise:
     - Iterate through the provided IDs.
     - Keep IDs that exist in `nodes` and whose node has non-empty `justifications`.

2. **Early exit**
   - If the filtered list is empty, return `[]`.

3. **Initialize batching state**
   - Create `all_results = []`.
   - Compute `total_batches` with ceiling division:
     ```python
     (len(belief_ids) + batch_size - 1) // batch_size
     ```

4. **Iterate batches**
   - For each slice `belief_ids[i:i + batch_size]`:
     - Print progress to `stderr`.
     - Format each belief using `format_belief_for_review(nid, nodes)`.
     - Join formatted beliefs with blank lines.
     - Build the review prompt via `REVIEW_PROMPT.format(beliefs=beliefs_text)`.

5. **Call the LLM**
   - Inside a `try` block:
     - Call `invoke_model(prompt, model=model, timeout=timeout)`.
     - Parse the response with `parse_review_response(response)`.
     - Extend `all_results` with parsed results.
     - Call `on_batch(all_results)` if provided.

6. **Handle per-batch failures**
   - If anything inside the `try` block raises an `Exception`, print a warning to `stderr` and continue to the next batch.
   - This means a failed batch does not abort the whole review.

7. **Return**
   - Return `all_results`.

## Side Effects

- Prints progress messages to `sys.stderr`:
  - `Reviewing batch X/Y (...)`
- Prints warnings to `sys.stderr` for failed batches:
  - `WARN: batch X failed: ...`
- Calls an external LLM through `invoke_model`.
- Calls the optional `on_batch` callback with the cumulative result list.
- Does not mutate `nodes`.
- Does not mutate the caller’s `belief_ids` list; it creates a new filtered list.
- May mutate `all_results` through the callback if the callback modifies the list.

## Error Handling

- Exceptions inside the per-batch `try` block are swallowed after logging:
  - `invoke_model` errors, including timeouts
  - `parse_review_response` errors
  - `on_batch` callback errors
- No retries are performed.
- A batch that returns unparseable JSON is not treated as a hard failure if `parse_review_response` returns `[]`; it simply contributes no results, and `on_batch` is still called.
- Errors before the `try` block are not caught:
  - malformed node data causing `format_belief_for_review` to raise
  - prompt formatting errors
  - invalid `batch_size` causing arithmetic issues
- `KeyboardInterrupt` and `SystemExit` are not caught because the code catches `Exception`, not `BaseException`.

## Usage Patterns

Typical usage:

```python
results = review_beliefs(network, model="claude", timeout=300)
```

To review a subset:

```python
results = review_beliefs(network, belief_ids=["b1", "b2"], batch_size=10)
```

To persist progress incrementally:

```python
def save_results(cumulative_results):
    write_json("review_results.json", cumulative_results)

results = review_beliefs(network, on_batch=save_results)
```

Caller obligations:

- Provide a valid `nodes` mapping.
- Provide a positive `batch_size`.
- Handle partial or empty results when batches fail.
- Do not assume returned IDs exactly match the requested batch.
- Be aware that `on_batch` receives a cumulative mutable list, not a per-batch delta.

## Dependencies

- `sys` for `stderr`.
- `invoke_model` from `.llm` for LLM calls.
- `REVIEW_BATCH_SIZE` and `REVIEW_PROMPT` from this module.
- `format_belief_for_review` from this module for prompt construction.
- `parse_review_response` from this module for response parsing and normalization.
- Indirectly, `json` through `parse_review_response`.

## Topics to Explore

- [file] `reasons/review.py` — Contains the review prompt, response parser, formatter, and batching logic.
- [function] `reasons/review.py:format_belief_for_review` — Defines exactly how claims, antecedents, outlists, and labels are serialized for the LLM.
- [function] `reasons/review.py:parse_review_response` — Normalizes LLM output and controls fallback defaults for missing review fields.
- [function] `reasons/llm.py:invoke_model` — Handles model invocation, timeout behavior, and provider-specific failures that `review_beliefs` catches per batch.
- [file] `tests/test_review.py` — Shows expected filtering, batching, callback, and failure behavior.

## Beliefs

- `review-beliefs-selects-in-nodes-by-default` — When `belief_ids` is `None`, only nodes with `truth_value == "IN"` and non-empty `justifications` are selected.
- `explicit-belief-ids-skip-in-filter` — When `belief_ids` is provided, the function filters only for node existence and non-empty `justifications`, not for `truth_value == "IN"`.
- `batch-failures-are-swallowed` — Exceptions inside the per-batch `try` block are logged to `stderr` and do not abort later batches.
- `on-batch-receives-cumulative-mutable-list` — The `on_batch` callback receives the same mutable `all_results` list after each processed batch.
- `review-beliefs-returns-normalized-result-dicts` — Returned items are normalized by `parse_review_response` with defaults for missing review fields.
