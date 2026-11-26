---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Understand why `api:` and `vertex:` calls are not included in `_record_cost` and what that means for `format_cost_summary()`.

## Why `api:`/`vertex:` bypass `_record_cost`, and what that does to `format_cost_summary()`

### The `_record_cost` call graph (all in `reasons/llm.py`)

`_record_cost` is defined once at `reasons/llm.py:103` and has exactly **three** call sites in production code:

| Line | Caller | Triggered by |
|---|---|---|
| `reasons/llm.py:145` | `_parse_cli_json` (gemini branch) | CLI `gemini` / `gemini:*` subprocess output |
| `reasons/llm.py:153` | `_parse_cli_json` (claude branch) | CLI `claude` / `claude:*` subprocess output |
| `reasons/llm.py:271` | `_invoke_openai` | `openai:*` Responses API calls |

Every one of those paths parses structured provider output for `usage`/`stats` and forwards token counts plus (for Claude CLI) a `total_cost_usd`. That's the whole mechanism.

### Where `api:`/`vertex:` diverge

In `invoke_model` (the dispatcher, around `reasons/llm.py:289`) the code short-circuits:

```python
if model.startswith("api:") or model.startswith("vertex:"):
    return _invoke_api(prompt, model, timeout)
```

`_invoke_api` (≈`reasons/llm.py:213`) builds a `ChatAnthropic`/`ChatAnthropicVertex`, calls `llm.invoke(prompt, config=config)`, and then **returns only `response.content`**. It never inspects `response.usage_metadata` (the LangChain field that carries input/output tokens) and never calls `_record_cost`. So nothing about an `api:`/`vertex:` call reaches the tracker:

- `_cost_tracker["calls"]` is not incremented
- `_cost_tracker["input_tokens"]` / `output_tokens` / `total_cost_usd` are not incremented
- No entry is created under `_cost_tracker["by_model"][model]` (entries are only synthesized inside `_record_cost`)

This isn't an intentional "API pricing is unknown" branch — there is simply no tracking code in that function.

### Subtle inconsistency: elapsed time *is* tracked

Look at the `finally:` block at the end of `invoke_model` (≈`reasons/llm.py:318–326`):

```python
finally:
    elapsed = time.monotonic() - t0
    with _cost_lock:
        _cost_tracker["elapsed_seconds"] += elapsed
        if model in _cost_tracker["by_model"]:
            _cost_tracker["by_model"][model]["elapsed_seconds"] += elapsed
```

This runs for *every* invocation, including `api:`/`vertex:`, because the dispatcher is inside the `try`. So `elapsed_seconds` **is** bumped for API calls, but `calls`, tokens, and cost are not. The tracking is half-done.

### Consequences for `format_cost_summary()` (`reasons/llm.py:87`)

```python
with _cost_lock:
    s = _cost_tracker
    if s["calls"] == 0:
        return ""
    ...
```

1. **API-only session → empty string.** If the whole run uses only `api:`/`vertex:` models, `calls` stays 0 and the function early-returns `""`. The elapsed time that *was* accumulated is silently discarded. Users see no cost line at all even though real tokens and money were spent.

2. **Mixed session → undercounted summary + inflated wall-time.** Suppose one `claude` CLI call plus one `api:claude-sonnet-4-20250514` call. The summary will report "1 call(s)" and only the CLI's tokens/cost, but `elapsed_seconds` includes *both* calls' latencies. The resulting string (e.g. `Cost: $0.0021 | 100 input + 50 output tokens | 1 call(s) | 12.4s`) is internally inconsistent — the single reported call cannot plausibly represent 12 seconds of token-priced work.

3. **`by_model` never has entries for `api:*` / `vertex:*`.** Anyone consuming `get_cost_summary()` (`reasons/llm.py:66`) for per-model attribution will see those models disappear entirely. Only CLI-model names and `openai:*` names appear.

4. **USD is already partial anyway.** Even within tracked paths, `gemini` (`reasons/llm.py:145`) and `openai` (`reasons/llm.py:271`) pass `0.0`; only the Claude CLI's `total_cost_usd` field (`reasons/llm.py:152`) supplies real currency. So `format_cost_summary()` is best read as "input/output token counts for CLI/OpenAI models, with USD only for the Claude CLI" — not a true end-to-end cost report.

5. **Callers are everywhere.** `format_cost_summary()` is imported and called ~25 times across `reasons/cli.py` (lines 1765, 2172, 2207, 2279, 2344, 2507, 2572, 2640, 2684, 2740, 2770, 2871, 2963, 3036, …). Every CLI command concludes with this panel. If a user's workflow runs on `api:` or `vertex:`, every one of those end-of-run panels will be empty or understated.

### Is the information available to fix it?

Yes — the LangChain `AIMessage` returned by `llm.invoke(...)` exposes `usage_metadata` (`input_tokens`, `output_tokens`, and sometimes `total_tokens`/`input_token_details.cache_read`), which is exactly what `_record_cost(model, input_tokens, output_tokens, cost_usd)` expects. The gap is purely that `_invoke_api` drops that field and returns `response.content` alone. `TestInvokeApi` in `tests/test_llm.py` mocks the response with only `.content`, so the test suite doesn't catch (or assert) any tracker mutation from the `api:`/`vertex:` path.

### What "the code currently means"

- `_record_cost` is a **CLI-and-OpenAI-only** accounting function; `api:` and `vertex:` are outside its scope.
- `format_cost_summary()` is therefore a **partial** summary: it reports real call counts only for subprocess and OpenAI paths; USD only for the Claude CLI; and it silently omits `api:`/`vertex:` tokens, calls, and currency while still absorbing their latency into `elapsed_seconds`.
- If a caller needs accurate accounting for `api:`/`vertex:`, they must either call `_record_cost` themselves or the fix belongs in `_invoke_api` around `return response.content`.

---

## Topics to Explore

- [function] `reasons/llm.py:_invoke_api` — The exact place a fix would go; inspect what `response.usage_metadata` actually contains for `ChatAnthropic` and `ChatAnthropicVertex` (fields differ slightly between the two adapters).
- [function] `reasons/llm.py:invoke_model` — The `finally:` block is where the mixed-signal behavior (elapsed counted, calls not) originates; worth understanding before changing any tracking logic.
- [file] `reasons/cli.py` — The ~25 call sites of `format_cost_summary()`; each is a UI surface affected by the under-reporting, and some may reset the tracker beforehand (`reset_cost_tracker`).
- [file] `tests/test_llm.py` — `TestCostTracker` and `TestInvokeApi` demonstrate the coverage gap: nothing asserts that `_invoke_api` mutates the tracker, so any fix will need new tests.
- [general] `langchain-usage-metadata` — The `AIMessage.usage_metadata` schema (`input_tokens`, `output_tokens`, `input_token_details.cache_read`) is what a correct implementation must parse to fill `_record_cost`.

## Beliefs

- `api-and-vertex-bypass-cost-tracker` — `_invoke_api` returns `response.content` without calling `_record_cost`, so `api:`/`vertex:` calls never increment tokens, calls, USD, or `by_model`.
- `format-cost-summary-returns-empty-for-api-only-sessions` — Because `format_cost_summary` (`reasons/llm.py:87`) early-returns `""` when `calls == 0`, a session that only uses `api:`/`vertex:` produces an empty cost line despite real spend.
- `elapsed-seconds-tracks-api-vertex-but-calls-does-not` — The `finally:` block of `invoke_model` increments `_cost_tracker["elapsed_seconds"]` for every invocation, while `calls` only increments inside `_record_cost`, so API latency is silently rolled into the summary displayed for mixed sessions.
- `record-cost-is-called-by-three-paths` — `_record_cost` (`reasons/llm.py:103`) is invoked only from `_parse_cli_json` (gemini at line 145, claude at line 153) and `_invoke_openai` (line 271).
- `cli-cost-has-real-usd-only-for-claude` — Only the Claude CLI’s `total_cost_usd` field (`reasons/llm.py:152`) supplies non-zero USD; gemini and `openai:*` paths pass `0.0` to `_record_cost`.
