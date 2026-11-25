---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How provider/model invocation is executed, retried, or batched in `_review`, since `parallel`, `on_batch`, and `timeout` all funnel there and drive the function's failure modes.

## How provider/model invocation is executed, retried, or batched in `_review`

`_review` is not defined in `reasons/review.py`. In `reasons/api.py:4084`, it is an import alias:

```python
from .review import review_beliefs as _review
```

So the relevant implementation is `review_beliefs` in `reasons/review.py`, called from `reasons/api.py:4148` and `reasons/api.py:4212`. Those call-site lines are truncated in the observations, so the full argument list is not visible.

### Batching behavior

`review_beliefs` performs batching sequentially:

- Default batch size is `REVIEW_BATCH_SIZE = 20`.
- If `belief_ids` is not provided, it selects derived `IN` nodes with at least one justification.
- If `belief_ids` is provided, it filters to IDs that exist and have justifications.
- It loops with:

```python
for i in range(0, len(belief_ids), batch_size):
    batch = belief_ids[i:i + batch_size]
```

- For each batch, it formats all beliefs into one prompt via `format_belief_for_review`, then calls:

```python
response = invoke_model(prompt, model=model, timeout=timeout)
```

- It parses the response with `parse_review_response`, extends `all_results`, and then calls `on_batch(all_results)` if provided.

There is no `parallel` parameter in the observed `review_beliefs` signature. The observed function is sequential. If a caller passes `parallel=...` directly to `review_beliefs`, Python would raise `TypeError`, unless an upstream wrapper strips it or a different `_review` alias is being used. The observations do not show any parallel implementation for `review_beliefs`.

### Provider/model invocation

`review_beliefs` delegates provider/model handling to `invoke_model` in `reasons/llm.py`.

`invoke_model` dispatches by model prefix:

- `api:<model>` / `vertex:<model>` → `_invoke_api`, using LangChain `ChatAnthropic` or `ChatAnthropicVertex`; timeout is passed to the adapter.
- `openai:<model>` → `_invoke_openai`, using the OpenAI Responses API with `timeout=timeout`.
- CLI models such as `claude`, `gemini`, `claude:<model>`, `gemini:<model>`, and `ollama:<model>` → `resolve_model_cmd`, then `subprocess.run(..., timeout=timeout)`.
- Missing CLI binaries raise `FileNotFoundError`.
- Non-zero CLI exits raise `RuntimeError`.
- CLI/API timeouts raise `subprocess.TimeoutExpired`.
- API dependency failures raise `ImportError`.

There is no retry logic in either `review_beliefs` or `invoke_model`. Each batch produces exactly one `invoke_model` attempt.

### Failure behavior

The key failure-handling block in `review_beliefs` is broad:

```python
try:
    response = invoke_model(prompt, model=model, timeout=timeout)
    results = parse_review_response(response)
    all_results.extend(results)
    if on_batch is not None:
        on_batch(all_results)
except Exception as e:
    print(f"  WARN: batch {batch_num} failed: {e}", file=sys.stderr)
```

Consequences:

- A failed batch does not stop the overall review. The loop continues to the next batch.
- Failed batches are omitted from `all_results`; there is no placeholder result.
- There is no retry for failed batches.
- `timeout` is per batch invocation, not a global review timeout. A review of `N` batches can take up to roughly `N * timeout` in the worst case before returning partial results.
- If `parse_review_response` cannot find a valid JSON array, it returns `[]` rather than raising. That means malformed LLM output is treated as a successful batch with zero results; `on_batch` is still called.
- If `on_batch` itself raises, the exception is caught by the same `except Exception` block. However, `all_results` has already been extended before the callback runs, so the batch results are present even though the batch is reported as failed.
- `on_batch` receives the same mutable `all_results` list each time, not a copy. Callbacks that store the reference may see later batches mutate previously observed results.

A separate path, `classify_defeat_reason` in `reasons/review.py`, also calls `invoke_model` with a timeout and swallows exceptions to return `""`, but it is not part of `review_beliefs`.

### Gaps in the observations

- The full call sites at `reasons/api.py:4148` and `reasons/api.py:4212` are truncated, so we cannot confirm whether `on_batch`, `parallel`, or other arguments are forwarded there.
- `reasons/api.py:4196` and `:4254` show other `_review` aliases (`review_justifications`, `review_premises`), but their implementations are not included.
- The `grep_review_parallel_on_batch_timeout` result reports zero matches, but the raw `reasons/review.py` content clearly contains `on_batch` and `timeout`. I treated the raw file content as authoritative.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full call sites around lines 4148 and 4212 to confirm whether `on_batch`, `parallel`, and other arguments are passed to `_review`.
- [function] `reasons/llm.py:invoke_model` — Trace provider dispatch, timeout translation, and absence of retry across CLI, Anthropic API, Vertex, and OpenAI paths.
- [file] `reasons/review.py` — Review batching, `parse_review_response`, and `on_batch` semantics in `review_beliefs`.
- [general] `review-callback-contract` — Determine how callers consume the cumulative `all_results` list passed to `on_batch` and whether they copy it.
- [general] `parallel-review-implementation` — Confirm whether any parallel review path exists outside the observed `review_beliefs` signature.

## Beliefs

- `review-beliefs-has-no-retry` — `review_beliefs` invokes `invoke_model` once per batch and catches exceptions without retrying or re-raising.
- `review-batches-are-sequential` — `review_beliefs` processes batches with a single `for` loop and has no `parallel` parameter in its observed signature.
- `on-batch-receives-cumulative-results` — `on_batch` is called with the same mutable `all_results` list after each successful batch.
- `timeout-is-per-batch` — `timeout` is forwarded to `invoke_model` for each batch, so it is not a global review timeout.
- `invalid-json-is-silent-empty-batch` — `parse_review_response` returns `[]` when no valid JSON array is found, so malformed LLM output produces zero results without raising.
