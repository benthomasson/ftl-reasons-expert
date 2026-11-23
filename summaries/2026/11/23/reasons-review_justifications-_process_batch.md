---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _process_batch in reasons/review_justifications.py

## Purpose

`_process_batch` is the private per-batch worker for justification review. It takes a batch of belief node IDs, formats those beliefs into the review prompt, sends the prompt to an LLM through `invoke_model`, parses the LLM response with `parse_justification_review`, emits a warning if nothing parseable came back, and returns the parsed result list.

It exists to isolate the single-batch LLM interaction from the batching, parallelism, aggregation, and error-reporting logic in `review_justifications`.

## Contract

### Preconditions

- `batch` is intended to be a non-empty list/slice of node IDs, usually a slice of `belief_ids` created by `review_justifications`.
- Every ID in `batch` should exist in `nodes` and be acceptable to `format_belief_for_review`.
- `nodes` should be the node mapping produced by `export_network()`, mapping `node_id -> node data`.
- `model` and `timeout` must be accepted by `invoke_model`.
- `REVIEW_JUSTIFICATIONS_PROMPT` must contain a `{beliefs}` format placeholder and otherwise have correctly escaped braces.

### Postconditions

- Returns a list of normalized review dictionaries from `parse_justification_review`, possibly empty.
- Does not mutate `batch`, `nodes`, `model`, or `timeout`.
- Does not update `all_results`; aggregation is the caller’s responsibility.
- Does not validate that returned result IDs match the input `batch`, nor that the number of results equals `len(batch)`.
- If parsing produces no results and `batch` is truthy, prints a warning to `sys.stderr`.

### Important invariant

The beliefs are rendered into the prompt in iteration order and joined with two newlines. The prompt asks the LLM to return one object per belief in the same order, but `_process_batch` does not enforce or verify that ordering or completeness.

## Parameters

- `batch`: Iterable/list of belief node IDs to review. In normal usage this is a list slice from `review_justifications`. If an ID is missing from `nodes`, `format_belief_for_review` may raise.
- `nodes`: Dictionary-like mapping of node ID to node data. It is only read.
- `model`: Model identifier passed through to `invoke_model`. No validation is performed here.
- `timeout`: Timeout value passed through to `invoke_model`. No validation is performed here.

Edge cases:

- `batch == []` does not short-circuit. The function builds an empty `beliefs_text`, formats the prompt, and still calls `invoke_model`.
- If `batch` is a one-shot iterator rather than a reusable list, it is consumed while building `beliefs_text`; the later `and batch` check may not behave like an emptiness check.
- If `nodes` lacks a requested ID, behavior depends on `format_belief_for_review`, but likely raises `KeyError`.

## Return Value

Returns whatever `parse_justification_review(response)` returns:

- A list of dictionaries with fields like `id`, `classification`, `required_antecedents`, `independent_antecedents`, and `comment`.
- `[]` if no parseable JSON array is found, or if the parse found no valid result objects.
- The function does not distinguish in its return value between “LLM returned nothing useful” and “LLM response was slightly malformed but parseable to zero results.”

Caller obligations:

- Handle `[]` results.
- Handle partial results, extra results, reordered results, or invalid IDs if correctness matters.
- Catch exceptions if calling directly, because `_process_batch` does not catch them itself.

## Algorithm

1. Iterate over IDs in `batch`.
2. For each ID, call `format_belief_for_review(nid, nodes)`.
3. Join the formatted belief strings with `"\n\n"`.
4. Substitute the combined text into `REVIEW_JUSTIFICATIONS_PROMPT` via `.format(beliefs=beliefs_text)`.
5. Call `invoke_model(prompt, model=model, timeout=timeout)`.
6. Pass the raw response to `parse_justification_review`.
7. If parsing returns no results and `batch` is truthy, print a warning to `sys.stderr`.
8. Return the parsed result list.

## Side Effects

- Performs an external LLM call through `invoke_model`, which may involve network I/O, latency, cost, authentication, and model-specific failures.
- Writes a warning to `sys.stderr` when a non-empty batch returns zero parseable results.
- Does not modify the database and does not mutate `nodes` or the batch.
- When `review_justifications` uses `parallel > 0`, `_process_batch` runs inside a `ThreadPoolExecutor`. This means `invoke_model` must be safe for concurrent calls if parallel execution is used.

## Error Handling

`_process_batch` deliberately does not catch exceptions. These can propagate:

- `format_belief_for_review` errors, such as missing IDs or malformed node data.
- Prompt formatting errors if the prompt template is malformed.
- `invoke_model` errors, such as network failures, timeouts, authentication errors, or invalid model arguments.
- `parse_justification_review` errors for unexpected response types, e.g. `response is None`.

In normal usage, `review_justifications` catches exceptions around `_process_batch` and logs a batch-level warning. That means a failed batch is skipped and contributes no results.

`parse_justification_review` is defensive about malformed JSON: it catches `json.JSONDecodeError`, scans for a parseable JSON array, and skips invalid individual items. However, `_process_batch` itself only warns when the parsed result list is completely empty for a non-empty batch. It does not warn about partial parses.

## Usage Patterns

`_process_batch` is called internally by `review_justifications`:

- Sequentially for each batch when `parallel <= 0`.
- Through `ThreadPoolExecutor` when `parallel > 0`.

Callers normally do not call `_process_batch` directly because it is private and omits validation, retries, and aggregation. The caller is responsible for:

- Creating valid batches of IDs.
- Ensuring IDs exist in `nodes`.
- Aggregating results across batches.
- Logging failed batches.
- Optionally validating that returned IDs correspond to requested beliefs.

## Dependencies

- `format_belief_for_review` from `reasons.review` for rendering each belief.
- `invoke_model` from `reasons.llm` for the LLM call.
- `parse_justification_review` from the same module for response parsing.
- `REVIEW_JUSTIFICATIONS_PROMPT` from the same module as the prompt template.
- `sys.stderr` for warning output.

## Assumptions Not Enforced by the Type System

- `batch` is assumed to contain IDs present in `nodes`.
- `nodes[nid]` is assumed to have the structure expected by `format_belief_for_review`.
- `invoke_model` is assumed to return a string suitable for `parse_justification_review`.
- The LLM is assumed to be capable of producing parseable JSON, but the code tolerates failure by returning `[]`.
- Parsed result IDs are assumed to be trustworthy enough for the caller; `_process_batch` does not check them against `batch`.
- A non-empty `batch` is assumed for normal operation; empty batches are not specially short-circuited.
- In parallel mode, `invoke_model` is assumed to be thread-safe.

## Topics to Explore

- [function] `reasons/review.py:format_belief_for_review` — Defines exactly how each belief node becomes prompt text.
- [function] `reasons/llm.py:invoke_model` — Controls model invocation, timeout behavior, retries, and provider-specific failures.
- [function] `reasons/review_justifications.py:parse_justification_review` — Implements the defensive JSON extraction and normalization used here.
- [function] `reasons/review_justifications.py:review_justifications` — Handles batching, parallelism, aggregation, and swallowing batch-level exceptions.
- [file] `tests/test_review_justifications.py` — Shows expected parsing, batching, and warning behavior around this review flow.

## Beliefs

- `process-batch-does-not-validate-result-ids` — `_process_batch` returns parsed results without checking that their `id` fields belong to the input `batch`.
- `process-batch-propagates-exceptions` — `_process_batch` does not catch formatting, prompt, model-invocation, or parsing exceptions.
- `empty-parse-warning-requires-nonempty-batch` — A warning is printed to stderr only when parsing returns no results and `batch` is truthy.
- `process-batch-invokes-model-for-empty-batch` — An empty `batch` still causes `invoke_model` to be called with an empty beliefs section.
- `process-batch-preserves-input-order-in-prompt` — Beliefs are rendered in iteration order and joined with two newlines before prompt formatting.
