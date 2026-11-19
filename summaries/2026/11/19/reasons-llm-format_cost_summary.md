---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: format_cost_summary in reasons/llm.py

## Purpose

`format_cost_summary()` produces a concise, human-readable one-line summary of the module-level LLM usage tracker (`_cost_tracker`). It is intended for logging or displaying aggregate token/cost information after one or more LLM calls have been recorded.

It does **not** provide per-model breakdowns; that data exists in `_cost_tracker["by_model"]` but is ignored here.

## Contract

**Preconditions**
- Module globals `_cost_lock` and `_cost_tracker` must exist and be initialized.
- `_cost_tracker` must contain the expected keys:
  - `calls`
  - `input_tokens`
  - `output_tokens`
  - `total_cost_usd`
  - `elapsed_seconds`
- The corresponding values must support the operations used below: equality/comparison to `0`, integer-style `:,` formatting for tokens, and float-style `.4f`/`.1f` formatting for cost/time.
- The caller must not already hold `_cost_lock` in the same thread, because `threading.Lock` is not reentrant.

**Postconditions**
- Returns `""` when `_cost_tracker["calls"] == 0` at the time of the locked read.
- Otherwise returns a string beginning with `"Cost: "`, followed by one or more parts joined by `" | "`.
- The token part and call-count part are always included when `calls != 0`.
- The dollar part is included only when `total_cost_usd > 0`.
- The elapsed-time part is included only when `elapsed_seconds > 0`.
- The function does not mutate `_cost_tracker`.

**Invariants**
- All reads of `_cost_tracker` happen while `_cost_lock` is held.
- The lock is released on every return path via the `with` statement.
- `by_model` is never read or formatted by this function.

## Parameters

This function takes no parameters.

## Return Value

Returns a `str`.

Possible shapes:

```text
""
```

when no calls have been recorded.

Otherwise, examples:

```text
Cost: $0.1234 | 1,234 input + 567 output tokens | 2 call(s) | 3.4s
Cost: 1,234 input + 567 output tokens | 2 call(s)
Cost: $0.0100 | 0 input + 0 output tokens | 1 call(s)
```

Callers must handle the empty-string case. They should not assume the dollar amount or elapsed-time segment is always present.

## Algorithm

1. Acquire `_cost_lock`.
2. Bind `s = _cost_tracker`.
3. If `s["calls"] == 0`, return `""` immediately.
4. Initialize `parts = []`.
5. If `s["total_cost_usd"] > 0`, append a dollar amount formatted to four decimal places:
   - `f"${s['total_cost_usd']:.4f}"`
6. Append the token counts with thousands separators:
   - `f"{s['input_tokens']:,} input + {s['output_tokens']:,} output tokens"`
7. Append the call count:
   - `f"{s['calls']} call(s)"`
8. If `s["elapsed_seconds"] > 0`, append elapsed seconds formatted to one decimal place:
   - `f"{s['elapsed_seconds']:.1f}s"`
9. Join all parts with `" | "`, prefix with `"Cost: "`, and return.

## Side Effects

- Acquires and releases `_cost_lock`.
- May block if another thread holds the lock.
- Reads shared mutable global state.
- Does not mutate tracker state.
- Does no I/O directly.

## Error Handling

There is no explicit validation or exception handling.

Possible failures:
- `KeyError` if expected `_cost_tracker` keys are missing.
- `TypeError`/`ValueError` if values have incompatible types for comparison or formatting.
- Deadlock if called by a thread that already holds `_cost_lock`, because `threading.Lock` is non-reentrant.
- If an exception occurs during formatting, the `with _cost_lock` block still releases the lock.

The function does not swallow errors or provide a fallback string.

## Usage Patterns

Typical usage:

```python
summary = format_cost_summary()
if summary:
    print(summary)
```

Callers should:
- Treat `""` as “no recorded LLM calls”.
- Avoid calling this while holding `_cost_lock`.
- Use `get_cost_summary()` when structured aggregate data or per-model details are needed.
- Call `reset_cost_tracker()` to clear accumulated state.

Important behavioral assumption: `calls` is the gate for showing anything. If `elapsed_seconds` is non-zero but `calls == 0`, the function returns `""`. This can happen because `invoke_model()` records elapsed time in a `finally` block even when no cost/token record was produced, such as failed calls, `ollama:` calls, or API/Vertex paths that do not call `_record_cost()`.

## Dependencies

Internal:
- `_cost_lock`
- `_cost_tracker`

The function indirectly depends on `_record_cost()` to populate `calls`, token counts, and cost, and on `reset_cost_tracker()` to clear state.

External/built-in:
- `threading.Lock`
- Python f-string formatting with `:,`, `.4f`, and `.1f`

## Assumptions Not Enforced by Types

- `_cost_tracker` has exactly the expected schema.
- `calls` is the authoritative indicator that any usage data exists.
- `input_tokens`, `output_tokens`, and `calls` are integer-like.
- `total_cost_usd` and `elapsed_seconds` are numeric and comparable to `0`.
- A zero cost is treated as “omit the cost component”, not “show $0.0000”.
- Shared reads are only safe if all writes also use `_cost_lock`.
- The function is not reentrant.

## Topics to Explore

- [function] `reasons/llm.py:_record_cost` — Shows which calls increment token/cost stats and how `by_model` is maintained.
- [function] `reasons/llm.py:invoke_model` — Explains why `elapsed_seconds` can advance even when `calls` does not.
- [file] `tests/test_llm.py` — Likely contains formatting and tracker-reset expectations for this summary.
- [general] `cost-tracker-thread-safety` — Covers `_cost_lock` usage, contention, and the non-reentrant deadlock risk.
- [function] `reasons/llm.py:get_cost_summary` — Structured alternative that deep-copies the tracker and preserves per-model data.

## Beliefs

- `format-cost-summary-returns-empty-with-zero-calls` — When `_cost_tracker["calls"] == 0`, the function returns `""` and suppresses all other fields, including any nonzero elapsed time.
- `format-cost-summary-omits-zero-cost` — The dollar amount appears only when `total_cost_usd > 0`; zero or negative cost produces no cost segment.
- `format-cost-summary-reads-under-cost-lock` — The function reads `_cost_tracker` while holding `_cost_lock` and does not mutate tracker state.
- `format-cost-summary-requires-cost-tracker-schema` — It assumes numeric `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, and `elapsed_seconds` keys exist and does not validate them.
- `format-cost-summary-ignores-per-model-breakdown` — It formats only aggregate tracker fields and never reads `by_model`.
