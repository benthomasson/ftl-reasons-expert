---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: In `dual` mode `timeout` is per-call, not total; callers of `ask(..., dual=True)` may inadvertently block for minutes

# Why `dual=True` multiplies the timeout budget

## Short version

`ask()` takes a single `timeout` parameter (default `300` seconds) and its signature makes it look like a global budget. In `dual` mode that parameter is passed *unchanged* to each of several independent LLM invocations. The result is that one `ask(..., dual=True, timeout=300)` call can block for **3× (`simple=True`) up to ~5× (`simple=False`) the requested timeout** — i.e. 15–25+ minutes from a caller's point of view.

The docstring already admits this (`"Timeout applies per call."`, `reasons/ask.py` line ~399), but the API shape doesn't enforce or surface it.

## Where the multiplier comes from

### 1. `ask()` signature and dual branch

`reasons/ask.py:383`:

```python
def ask(question, db_path="reasons.db", timeout=300, no_synth=False, format=None,
        model="claude", simple=False, sources_db=None, natural=False, dual=False,
        mcp_servers=None):
```

The `dual` docstring text (lines 394–400) is explicit about call counts:

> *"dual: run TMS and FTS RAG in separate calls, then merge the two answers in a third call. Requires sources_db. Makes 3 LLM calls with simple=True (1 TMS + 1 FTS + 1 merge), or up to 5 with simple=False (up to 3 TMS tool-loop rounds + 1 FTS + 1 merge). **Timeout applies per call.**"*

The guard at `reasons/ask.py:403-404` requires `sources_db`, so this only bites callers who have properly enabled `--full-sources` — which makes it easy to miss.

### 2. Each stage independently calls `invoke_model(..., timeout=timeout)`

The three helper entry points all declare their own `timeout=300` and pass it straight down:

- `_fts_rag_answer(question, sources_db, model="claude", timeout=300)` — `reasons/ask.py:~349`
  - ends with `return invoke_model(prompt, model=model, timeout=timeout).strip()`
- `_merge_answers(question, answer_tms, answer_fts, model="claude", timeout=300)` — `reasons/ask.py:~365`
  - same pattern
- The TMS path uses `MAX_ITERATIONS = 3` (`reasons/ask.py`, just above `NO_BELIEFS_MSG`) and calls `invoke_model` inside the loop. There is also a final "extra synthesis" call after a tool-call round — evidenced by `test_final_tool_call_triggers_extra_synthesis` in `tests/test_ask.py`, which asserts `calls[0] == 4` on the non-simple path.

So the TMS stage alone can burn `4 × timeout` seconds; add FTS and merge for the total.

### 3. `invoke_model()` does not aggregate

`reasons/llm.py:invoke_model` (~line 285) forwards the timeout to the transport:

```python
result = subprocess.run(cmd, input=prompt, capture_output=True,
                        text=True, timeout=timeout, env=env)
```

…or to `ChatAnthropic(timeout=float(timeout))`, `ChatAnthropicVertex(request_timeout=float(timeout))`, or `client.responses.create(..., timeout=timeout)`. Every one of those is a **per-request** deadline. There is no parent deadline/deadline-propagation object anywhere in the call chain.

### 4. Concrete worst-case math (default `timeout=300`)

| Mode | Stages | Worst-case wall clock |
|---|---|---|
| `ask(dual=True, simple=True)` | TMS + FTS + merge = 3 calls | **~15 min** |
| `ask(dual=True, simple=False)` | up to 4 TMS + FTS + merge = up to 6 calls (docstring says 5) | **~25–30 min** |

This aligns with `tests/test_ask.py:792` (`test_dual_calls_three_times`) and the tests at 808/818/837 that drive `ask(..., dual=True)` end-to-end.

### 5. Timeouts don't short-circuit the multiplier

When `invoke_model` raises `subprocess.TimeoutExpired`, the non-dual TMS path catches it and returns whatever beliefs were already gathered (`test_timeout_returns_search_results`, `test_timeout_on_empty_returns_no_beliefs_message`). But those are *per-stage* recoveries. A `dual` run that eats 12 minutes on three slow calls and then recovers on the fourth still ate 12 minutes — there's no "give up after 5 minutes total" mechanism.

## Why callers get bitten

- **Signature shape**: `timeout=300` looks like a single budget, not a per-request parameter.
- **Docstring depth**: the caveat lives in the middle of a long `Args:` block; the CLI wiring at `reasons/cli.py:1377` simply forwards `dual=args.dual` without touching `timeout`, so a user doing `reasons ask ... --dual --full-sources` gets the default 300 multiplied.
- **No deadline propagation**: nothing in `ask` → `_fts_rag_answer` / `_merge_answers` / TMS loop carries a "remaining budget" — each stage starts with a fresh `timeout`.

## Practical mitigations for callers

1. Compute a per-call timeout: `call_timeout = total_budget // 3` (simple) or `// 5` (non-simple) before calling `ask(..., dual=True)`.
2. Wrap `ask()` in a caller-side wall-clock guard (thread + `join(total_budget)`), since the function itself has no aggregate deadline.
3. Prefer `simple=True` when using dual, since it caps the call count at 3.

## What the observations don't show

The body of `ask()` after the docstring (`reasons/ask.py` lines ~410–566) is *not* included in the read. The per-call fan-out is inferred from (a) the docstring, (b) the helper signatures `_fts_rag_answer`/`_merge_answers` which both hardcode `timeout=300`, and (c) the dual tests. If you want certainty that the outer `timeout` is forwarded verbatim to all three helpers, that body is the file to read directly.

## Topics to Explore

- [file] `reasons/ask.py` — Read the `ask()` body (lines 410–566) to confirm exactly how `timeout` is threaded into the TMS loop, `_fts_rag_answer`, and `_merge_answers`.
- [function] `reasons/ask.py:ask` — Trace the `dual` branch and the `MAX_ITERATIONS` loop to count the maximum number of `invoke_model` calls on each code path.
- [function] `reasons/llm.py:invoke_model` — Confirm how `timeout` maps to subprocess vs. LangChain vs. OpenAI under each model prefix, especially whether any adapter ignores the argument.
- [file] `tests/test_ask.py` — The `TestAskDual` class at ~line 790 exercises `dual=True`; check whether it passes a non-default `timeout` or asserts on call counts/timeouts.
- [file] `reasons/cli.py` — Around line 1377 (`dual=args.dual`), confirm whether the CLI exposes a `--timeout` and whether it's forwarded blindly into `ask()` without a per-call budget adjustment.

## Beliefs

- `ask-dual-timeout-is-per-call` — `ask(..., dual=True, timeout=T)` forwards `T` unchanged to each of the TMS, FTS RAG, and merge `invoke_model` calls, so total wall-clock can reach N×T.
- `dual-simple-triggers-three-llm-calls` — With `simple=True` and `dual=True`, `ask()` makes exactly 3 `invoke_model` calls (TMS + FTS + merge), as asserted by `tests/test_ask.py:792`.
- `tms-path-iterates-up-to-max-iterations-plus-one` — The TMS stage loops up to `MAX_ITERATIONS = 3` tool-call rounds and can add one extra synthesis call, for up to 4 LLM invocations (per `test_final_tool_call_triggers_extra_synthesis`).
- `invoke-model-forwards-timeout-to-transport` — `invoke_model()` passes its `timeout` argument directly to `subprocess.run(timeout=...)` for CLI models and to the LangChain/OpenAI adapter's own timeout parameter for API models.
- `dual-requires-sources-db` — `ask(..., dual=True)` raises `ValueError("--dual requires --full-sources")` when `sources_db` is falsy (`reasons/ask.py:403-404`).
- `ask-has-no-aggregate-deadline` — Neither `ask()` nor any of its helpers constructs a shared deadline object; each stage starts with a fresh `timeout`, so per-stage multiplication is unavoidable without caller-side enforcement.
