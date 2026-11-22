---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Whether other modules use cumulative or per-batch callbacks, and how the CLI consumes them

## Summary

The observed modules use **per-batch / per-write callbacks**, not cumulative callbacks. There is no evidence of a cumulative callback that fires once with accumulated results across all batches.

The CLI, based on the observations, does **not directly consume or expose these callbacks**. A grep of `reasons/cli.py` for `callback|stream|on_token|on_chunk|on_delta` returned zero matches. The `--stream` CLI flags are unrelated: they filter by workstream tag (`reasons/cli.py:3573`, `reasons/cli.py:3911`).

## Evidence

- `reasons/review_justifications.py:123` documents:
  - `on_batch: Optional callback after each batch.`
  - This is explicitly per-batch invocation semantics.

- `reasons/pubsub.py:38-47` defines `subscribe(callback, db_path=None)`.
  - The docstring at `reasons/pubsub.py:42` says:
    - `callback: Called with a list of ChangeEvent after each write`
  - The callback is registered per subscriber at `reasons/pubsub.py:47`.
  - This is a per-write batch callback, not a cumulative history callback.

- `tests/test_review.py:561` says:
  - `"on_batch callback writes partial report after each batch."`
  - This confirms the review flow expects per-batch callbacks.

- `tests/test_propose_update.py:439` and `tests/test_review_premises.py:110` both contain `test_on_batch_callback`, consistent with per-batch callback behavior in those modules.

- `tests/test_repair_premises.py:142-155` tests `on_result`:
  - `on_result=lambda r: callbacks.append(len(r))`
  - `assert callbacks == [1]`
  - This shows a single on-result-style callback invocation with one result, not a growing cumulative sequence.

- `reasons/llm.py:236` sets:
  - `config["callbacks"] = [handler]`
  - This is a LangChain/Langfuse callback passed to a single `llm.invoke(...)` call in `_invoke_api`.
  - It is per LLM invocation tracing, not an application-level cumulative batch callback.
  - `tests/test_llm.py:319-334` confirms a mock handler is passed through `call_config["callbacks"]`.

- No matches were found for `cumulative`, `per_batch`, `per-batch`, `batch_callback`, `on_token`, `on_chunk`, or `on_delta` in the callback-related search. The only `stream` matches are unrelated workstream/tag filtering or HTTP content-type usage.

## CLI consumption

`inspect_cli_callback_handling` found **zero** matches for `callback|stream|on_token|on_chunk|on_delta` in `reasons/cli.py`. `inspect_derive_callback_handling` also found zero matches.

So the CLI appears to **not consume these callbacks directly**. It does not appear to accept callback arguments or forward them from CLI commands. The CLI may call higher-level functions whose callback parameters default to `None`, but the observations do not show the CLI passing `on_batch`, `on_result`, or similar callbacks.

What is missing to fully prove this:
- The actual definitions and call sites of `on_batch` / `on_result` in review/propose/repair modules.
- The full CLI command implementations that invoke those modules.
- Function signatures for the review/propose/repair entry points to see whether callbacks are optional and whether CLI code passes them.

## Topics to Explore

- [file] `reasons/review_justifications.py` — Contains the `on_batch` documentation at line 123; inspect implementation and call sites to confirm callback payload semantics.
- [function] `reasons/pubsub.py:subscribe` — Registers callbacks invoked after each write with a list of `ChangeEvent`.
- [file] `reasons/cli.py` — Verify whether any CLI command forwards `on_batch` / `on_result`; distinguish workstream `--stream` flags from streaming callbacks.
- [function] `reasons/llm.py:_invoke_api` — Shows the Langfuse callback is attached to a single `llm.invoke` call, not to batch accumulation.
- [general] `callback-call-sites` — Search for `on_batch=` and `on_result=` across CLI and review/propose/repair modules to see actual wiring.

## Beliefs

- `review-on-batch-is-per-batch` — `reasons/review_justifications.py:123` documents `on_batch` as a callback invoked after each batch.
- `pubsub-callback-is-per-write` — `reasons/pubsub.py:42` documents subscriber callbacks as receiving a list of `ChangeEvent` after each write.
- `llm-callback-is-per-invocation` — `reasons/llm.py:236` passes a Langfuse callback handler into a single `llm.invoke` call.
- `cli-has-no-direct-callback-handling` — `reasons/cli.py` has zero matches for callback or token/chunk streaming terms in the supplied observations.
- `no-cumulative-callback-evidence` — The observations show per-batch/per-write callbacks and no cumulative callback implementation or usage.
