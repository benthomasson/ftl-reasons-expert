---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _record_cost in reasons/llm.py

# `_record_cost` — LLM Cost/Usage Accumulator

## Purpose

`_record_cost` is a private, thread-safe accumulator that folds the outcome of a single LLM invocation (token counts + dollar cost) into two nested counters:

- A **global** counter (`_cost_tracker`) with aggregate totals across all calls.
- A **per-model** counter (`_cost_tracker["by_model"][model]`) that mirrors the same schema for each distinct model identifier.

It exists so that any code path that successfully learns usage stats from a provider (CLI JSON parsing, OpenAI Responses API, etc.) can funnel it through one atomic, mutex-protected update — and so callers can later retrieve a consistent snapshot via `get_cost_summary()`.

## Contract

**Preconditions**
- `model` is a `str` that is hashable / usable as a dict key (this is the model identifier, e.g. `"claude"`, `"claude:sonnet"`, `"gemini:gemini-2.5-flash"`, `"openai:gpt-4o"`).
- `input_tokens` and `output_tokens` are numeric (int expected; not enforced).
- `cost_usd` is a float (may be `0.0` for providers that don't report cost, e.g. Gemini CLI, OpenAI).
- `_cost_tracker` and `_cost_lock` must already exist (module-level globals).

**Postconditions**
- `_cost_tracker["calls"]` is incremented by exactly 1.
- `_cost_tracker["input_tokens"]` / `"output_tokens"` / `"total_cost_usd"` are incremented by the arguments.
- `_cost_tracker["by_model"][model]` exists after the call (created with a zero-initialized dict if absent) and has all of `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `elapsed_seconds` incremented/applied for the four usage fields.
- No other keys of `_cost_tracker` are touched (notably `elapsed_seconds` at the top level is **not** modified here — it's handled by `invoke_model`'s `finally`).

**Invariants**
- All mutations occur while holding `_cost_lock`, so concurrent callers never see partial updates and per-model dicts never race on creation.
- The per-model schema is a subset of the top-level schema (top level additionally has `by_model`); the two must stay in sync if new fields are added.
- Accumulation is strictly additive — there is no per-call reset. The only reset path is `reset_cost_tracker()`.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `model` | `str` | Model identifier used as the sub-key in `by_model`. | Must be hashable. Callers pass raw user-facing model strings, so two logically-equivalent models with different strings (e.g. `"claude"` vs `"claude:"` suffix variants) are tracked separately. |
| `input_tokens` | `int` | Prompt/input token count for this call. | Gemini path aggregates `"input"` across sub-models; Claude path sums `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`. No validation — negative/float values would silently corrupt totals. |
| `output_tokens` | `int` | Completion/output token count for this call. | Gemini uses `"candidates"`; Claude uses `usage["output_tokens"]`. |
| `cost_usd` | `float` | Dollar cost of this single call. | `0.0` for Gemini and OpenAI (they don't surface cost in the parsed response); real value from Claude CLI's `total_cost_usd`. |

## Return Value

Returns `None` implicitly. Callers are expected to rely on the side effect, then read aggregated state later via `get_cost_summary()` / `format_cost_summary()`. There is no return value to check and no error signalling.

## Algorithm

1. Acquire `_cost_lock` (blocking).
2. Increment the four aggregate counters on `_cost_tracker`.
3. If `model` is not yet a key in `_cost_tracker["by_model"]`, insert a fresh dict with zeroed `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, and `elapsed_seconds`.
4. Bind `m = _cost_tracker["by_model"][model]` and increment `m["calls"]`, `m["input_tokens"]`, `m["output_tokens"]`, `m["total_cost_usd"]`.
5. Exit the `with` block (releases the lock).

Notice the "read-then-maybe-create-then-update" pattern is safe precisely because the whole sequence is under one lock — otherwise the `if model not in ...` check would be a classic check-then-act race.

## Side Effects

- **Global state mutation**: `_cost_tracker` dictionary (two nested levels) is modified in place.
- **Lock acquisition**: briefly contends with `_cost_lock`, which is also used by `reset_cost_tracker`, `get_cost_summary`, `format_cost_summary`, and the `finally` block of `invoke_model`.
- **No I/O**, no logging, no exceptions raised for normal flows.

## Error Handling

There is **no explicit error handling**. Errors are only the implicit kind:
- If `model` is unhashable (e.g. a list), the `model not in _cost_tracker["by_model"]` line raises `TypeError`.
- If token counts or cost are non-numeric, `+=` raises `TypeError`.
- If a caller passes `None` for `cost_usd`, `0.0 += None` raises `TypeError`.

Because callers `_parse_cli_json` and `_invoke_openai` extract these values from already-parsed JSON/API responses, they typically default to `0` / `0.0` — so failures here would signal a programming bug, not transient runtime noise. Nothing is swallowed.

## Usage Patterns

- Called from `_parse_cli_json` after successfully parsing `claude` or `gemini` CLI JSON output.
- Called from `_invoke_openai` after assembling the response text.
- **Not** called by:
  - `_invoke_api` (`api:` / `vertex:` LangChain paths) — these never record tokens or cost.
  - The `ollama:` CLI branch of `invoke_model` — that path returns raw text before reaching `_parse_cli_json`.
- Callers must call it at most once per invocation (otherwise usage is double-counted). There is no dedup.
- `elapsed_seconds` is intentionally not touched here — it is written in `invoke_model`'s `finally`, which is why the per-model dict includes that field but this function never increments it.

The key non-obvious asymmetry: **only Claude CLI, Gemini CLI, and OpenAI paths produce tracked stats**. Any repo-level reporting using `get_cost_summary()` will silently omit `api:`, `vertex:`, and `ollama:` usage.

## Dependencies

- `threading.Lock` — the `_cost_lock` global.
- Module-level `_cost_tracker` dict.
- No external libraries; standard library only.
- Relies on callers (`_parse_cli_json`, `_invoke_openai`) committing already-normalized numbers; there is no schema validation inside.

### Assumptions not enforced by the type system

- That `model` values are canonicalized so semantically-equal models don't fragment across dict keys.
- That token counts are non-negative integers (floats/negatives silently corrupt totals).
- That the identical schema is used for both top-level and per-model dicts — adding a field to one without the other will desync `get_cost_summary()` output.
- That no caller invokes `_record_cost` for a model that then never appears in `invoke_model`'s `finally` (or vice versa) — otherwise `elapsed_seconds` will stay at `0.0` forever for that model.

## Topics to Explore

- [function] `reasons/llm.py:_parse_cli_json` — The primary caller; responsible for extracting token counts and cost from CLI JSON, and for the model-prefix branching that determines what gets recorded.
- [function] `reasons/llm.py:invoke_model` — The `finally` block updates `elapsed_seconds`; also shows which model prefixes bypass `_record_cost` entirely.
- [function] `reasons/llm.py:get_cost_summary` — Demonstrates the read-side contract (deep copy under lock) that `_record_cost`'s writes feed.
- [file] `tests/test_llm.py` — Check how the per-model schema and thread safety are actually asserted.
- [general] `cost-tracker-coverage-gaps` — Reason about why `api:`, `vertex:`, and `ollama:` models are currently absent from `by_model` and whether that's intentional.

## Beliefs

- `cost-tracker-requires-cost-lock` — Every mutation and read of `_cost_tracker` occurs inside a `with _cost_lock:` block.
- `record-cost-produces-per-model-entry` — After the first call for a given `model`, `_cost_tracker["by_model"][model]` is populated with fields `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, and `elapsed_seconds`.
- `record-cost-is-additive-not-idempotent` — Repeated invocations accumulate rather than replace; only `reset_cost_tracker()` clears state.
- `api-and-vertex-paths-skip-cost-tracking` — `_record_cost` is never invoked for `api:` / `vertex:` / `ollama:` models, so those models will not appear in `by_model` unless manually inserted.
- `elapsed-seconds-updated-outside-record-cost` — Per-model `elapsed_seconds` is initialized to `0.0` in `_record_cost` but only incremented in `invoke_model`'s `finally` block.
