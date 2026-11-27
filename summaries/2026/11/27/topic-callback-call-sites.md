---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search for `on_batch=` and `on_result=` across CLI and review/propose/repair modules to see actual wiring.

## How the callback wiring works

The codebase uses two callback contracts:

- **`on_batch`** — batch-oriented review/proposal functions. The callback fires after each completed LLM batch.
- **`on_result`** — per-item repair. The callback fires after each individual repair result.

The CLI creates the callbacks only when report writing is requested, the API forwards them, and the domain modules invoke them with the accumulated result list.

### Wiring summary

| Layer | Location | What it does |
|---|---|---|
| CLI | `reasons/cli.py:2398` | Defines `on_batch = (lambda results: _write_report(results, "partial")) if write_report else None` |
| CLI | `reasons/cli.py:2410` | Passes `on_batch=on_batch` into the API |
| CLI | `reasons/cli.py:2623` | Defines the same batch callback lambda |
| CLI | `reasons/cli.py:2632` | Passes `on_batch=on_batch` |
| CLI | `reasons/cli.py:2724` | Defines `on_result = (lambda results: _write_report(results, "partial")) if write_report else None` |
| CLI | `reasons/cli.py:2733` | Passes `on_result=on_result` |
| CLI | `reasons/cli.py:2811` | Passes `on_batch=on_batch`; the definition is not present in the search results |
| API | `reasons/api.py:4149, 4213, 4313, 4517` | Forwards `on_batch` to callee functions |
| API | `reasons/api.py:4421` | Forwards `on_result` together with `parallel=parallel` |
| Review | `reasons/review.py:217` | `review_beliefs(... batch_size=REVIEW_BATCH_SIZE, on_batch=None)` |
| Review justifications | `reasons/review_justifications.py:113` | `review_justifications(... on_batch=None, parallel=0)`; calls `on_batch(all_results)` after each batch |
| Review premises | `reasons/review_premises.py:129` | `review_premises(... on_batch=None, parallel=0)`; calls `on_batch(all_results)` after each batch |
| Propose update | `reasons/propose_update.py:292` | `propose_update(... batch_size=PROPOSE_UPDATE_BATCH_SIZE, on_batch=None)`; body not shown |
| Repair premises | `reasons/repair_premises.py:130` | `repair_premises(... parallel=0, on_result=None)`; calls `on_result(all_results)` after each premise |
| Repair orchestrator | `reasons/repair.py` | No `on_batch=` or `on_result=` matches in the search results |

### Runtime behavior

In `reasons/review_justifications.py`, the callback is invoked after each successful batch:

```python
all_results.extend(results)
if on_batch is not None:
    on_batch(all_results)
```

The same pattern appears in `reasons/review_premises.py`. This means the callback receives the **cumulative** result list, not just the most recent batch. The CLI lambda then writes a partial report from that cumulative list:

```python
lambda results: _write_report(results, "partial")
```

In parallel mode, callbacks are driven by `as_completed`, so partial reports may arrive in completion order rather than original batch order. A failed batch logs a warning and skips the callback for that batch.

In `reasons/repair_premises.py`, the per-item callback works similarly:

```python
result = _repair_one(...)
all_results.append(result)
if on_result is not None:
    on_result(all_results)
```

So `on_result` also receives the cumulative list after each premise result. In parallel mode it again follows completion order. If `_repair_one` catches an exception, it returns an `"action": "error"` result, which is still appended and passed to the callback.

### What is not fully visible

The observations do not include:

- The full `reasons/api.py` functions around lines `4149–4517`, so the exact API entry points forwarding each callback are not visible.
- The full `reasons/cli.py` command functions around lines `2398–2811`, so the command names and `write_report` setup are not visible.
- The body of `reasons/propose_update.py` after line `292`, so it is not confirmed whether `propose_update` calls `on_batch` with cumulative results or only the current batch.
- The body of `reasons/review.py` after line `217`, so the exact `review_beliefs` callback invocation is not shown, though sibling modules use the same batch callback pattern.
- Any callback wiring inside `reasons/repair.py`; the search found no `on_batch=` or `on_result=` matches there.
- The definition of `_write_report`, so the exact partial-report write semantics are not shown.

### Tests as usage examples

The test matches confirm the callback API is exercised directly:

- `tests/test_propose_update.py:449` — `on_batch=lambda results: batches_seen.append(len(results))`
- `tests/test_repair_premises.py:154` — `on_result=lambda r: callbacks.append(len(r))`
- `tests/test_review.py:579` — `on_batch=capture_on_batch`
- `tests/test_review_premises.py:124` — `on_batch=lambda r: batches.append(len(r))`

## Topics to Explore

- [function] `reasons/cli.py:_write_report` — Determines how partial reports are written and whether repeated cumulative writes overwrite or append.
- [file] `reasons/api.py` — Shows the exact API entry points that forward `on_batch` and `on_result` to the review/propose/repair modules.
- [function] `reasons/propose_update.py:propose_update` — Confirms whether `propose_update` invokes `on_batch` with cumulative results and how it handles batch failures.
- [function] `reasons/review.py:review_beliefs` — Confirms callback invocation order and parallel behavior for the main review path.
- [file] `reasons/repair.py` — Shows whether the repair orchestrator wires callbacks to `repair_premises` or handles repair differently.

## Beliefs

- `review-justifications-on-batch-receives-cumulative-results` — In `reasons/review_justifications.py`, `on_batch` is called with `all_results` after each successful batch, not only the current batch.
- `repair-premises-on-result-receives-cumulative-results` — In `reasons/repair_premises.py`, `on_result` is called with `all_results` after each individual premise result.
- `cli-partial-callback-requires-write-report` — In `reasons/cli.py`, the `on_batch` and `on_result` lambdas are `None` unless `write_report` is truthy.
- `api-forwards-callbacks-without-owning-semantics` — `reasons/api.py` lines `4149–4517` pass `on_batch`/`on_result` through to domain functions; callback semantics are implemented in the domain modules.
- `parallel-callback-order-follows-completion` — In the parallel code paths for `review_justifications`, `review_premises`, and `repair_premises`, callbacks fire in future-completion order via `as_completed`, not original input order.
