---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How the process-global `get_cost_summary()` is consumed and rendered elsewhere (e.g. CLI output) and why it must be reset between invocations.

## How `get_cost_summary()` is consumed and rendered

The cost data lives in a module-level `_cost_tracker` dict in `reasons/llm.py`. `_record_cost()` mutates that dict under `_cost_lock`, and both the structured and human-readable accessors read from it:

- `reset_cost_tracker()` at `reasons/llm.py:69` zeroes the tracker fields, including `by_model`.
- `get_cost_summary()` at `reasons/llm.py:80` returns a `copy.deepcopy(_cost_tracker)` while holding `_cost_lock`, so callers get a safe snapshot.
- `format_cost_summary()` immediately follows it in `reasons/llm.py` and renders the same global tracker as a human-readable string, returning `""` when `calls == 0`, otherwise something like `"Cost: $... | N input + M output tokens | K call(s) | Xs"`.

Observed consumption:

- Tests consume `get_cost_summary()` directly: `tests/test_llm.py:348`, `368`, `385`, `398`, `429`, `439`, `448`, and assert `get_cost_summary()["calls"] == 0` after reset at `tests/test_llm.py:406` and `418`.
- The CLI observed imports `reset_cost_tracker` and `format_cost_summary` at many command entry points, e.g. `reasons/cli.py:1666-1667`, `2213-2214`, `2353-2354`, `2514-2515`, `2581-2582`, `2693-2694`, `2779-2780`, `2920-2921`, and `2993-2994`, and calls `reset_cost_tracker()` immediately.
- Importantly, the observed `get_cost_summary` usages do **not** include direct CLI calls. The CLI path appears to render via `format_cost_summary()`, not via `get_cost_summary()` itself. The exact CLI print statements are not in the observations; `read_cli_relevant_sections` only returned the first 240 lines, and `inspect_cli_cost_output` returned no matches despite the import lines. So the exact rendering call site is missing from the provided evidence.

In short: `_record_cost()` updates process-global state; `get_cost_summary()` exposes a structured snapshot; `format_cost_summary()` renders the same state for terminal output; CLI commands import the renderer and reset helper.

## Why reset between invocations

Because `_cost_tracker` is process-global, it persists for the lifetime of the Python process. A CLI command or test is a logical invocation, but the process may execute several of them without restarting. Without `reset_cost_tracker()`, counters such as `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, and `by_model` would accumulate across previous invocations.

The observed reset pattern solves that:

- CLI commands reset at command start via `reasons/cli.py` imports such as `1666-1667`, `2213-2214`, etc.
- Tests reset before each cost assertion at `tests/test_llm.py:345`, `365`, `382`, `395`.

So reset is not about thread safety—`_cost_lock` already protects mutations and reads. It is about **logical isolation**: making the next summary reflect only the current invocation rather than all LLM calls made earlier in the same process.

## Topics to Explore

- [file] `reasons/llm.py` — Full cost-tracking implementation: `_record_cost`, `reset_cost_tracker`, `get_cost_summary`, and `format_cost_summary`
- [function] `reasons/llm.py:format_cost_summary` — The human-readable renderer imported by CLI commands
- [file] `reasons/cli.py` — Exact command wrappers that reset and print cost summaries around the observed import lines
- [file] `tests/test_llm.py` — Tests that assert reset behavior and cost-summary contents
- [general] `per-invocation-cost-isolation` — Pattern of resetting process-global trackers at command/test boundaries

## Beliefs

- `cost-tracker-is-process-global` — `_cost_tracker` is a module-level dict in `reasons/llm.py` shared by all LLM calls in the process.
- `get-cost-summary-returns-deep-copy` — `get_cost_summary()` returns a `copy.deepcopy` of `_cost_tracker` under `_cost_lock`.
- `format-cost-summary-renders-human-string` — `format_cost_summary()` reads `_cost_tracker` under `_cost_lock` and returns a `"Cost: ..."` string, or `""` when no calls have been recorded.
- `reset-cost-tracker-enables-per-invocation-isolation` — CLI call sites import `reset_cost_tracker` and call it immediately to prevent previous invocation costs from accumulating.
- `get-cost-summary-has-no-observed-cli-usage` — Observed `get_cost_summary()` usages are tests and documentation; CLI imports and uses `format_cost_summary` and `reset_cost_tracker` instead.
