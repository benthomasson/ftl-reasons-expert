---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `invoke_model` is thread-safe and how timeouts surface as the `error` action.

# `invoke_model`: Thread-Safety and Timeout Semantics

## Thread-Safety — Partially Safe

`invoke_model` (defined at `reasons/llm.py:275`) mixes a genuinely thread-safe cost tracker with an unsynchronized lazy singleton, so the honest answer is: **safe for the per-call work, unsafe for the Langfuse handler**.

### What is safe

Everything related to cost accounting is protected by a single module-level lock, `_cost_lock = threading.Lock()` (declared near the top of `reasons/llm.py`, alongside `_cost_tracker`):

- `reset_cost_tracker`, `get_cost_summary`, and `format_cost_summary` all acquire `_cost_lock`.
- `get_cost_summary` returns `copy.deepcopy(_cost_tracker)` inside the lock, so callers get a consistent snapshot rather than a live reference.
- `_record_cost` mutates `_cost_tracker` and its nested `by_model` entries under the same lock.
- The `finally:` block at the end of `invoke_model` (lines ~317–326) adds `elapsed_seconds` to both the global counter and per-model entry under `_cost_lock`.

Because writes to `_cost_tracker` are also wrapped, the counters do not get corrupted when callers in `reasons/ask.py`, `reasons/repair.py`, `reasons/contradictions.py`, etc. invoke `invoke_model` concurrently.

Per-call work is also stateless:

- Each call builds its own `cmd` (via `resolve_model_cmd`), `input`, and filtered `env` dict. `MODEL_COMMANDS` is a read-only module constant, so concurrent reads are fine.
- `subprocess.run(...)` (line ~305) opens its own pipes/file descriptors per invocation. In CPython this is safe for concurrent use.
- `_invoke_api` constructs a fresh `ChatAnthropic`/`ChatAnthropicVertex` per call (around line ~205), and `_invoke_openai` constructs a fresh `openai.OpenAI()` per call, so no client state is shared between threads.

### What is not safe

`_get_langfuse_handler` (lines ~156–168) is the one thread-safety hole:

```python
global _langfuse_handler, _langfuse_checked
if _langfuse_checked:
    return _langfuse_handler
_langfuse_checked = True          # set BEFORE doing the work
if not _HAS_LANGFUSE:
    return None
if not os.environ.get("LANGFUSE_PUBLIC_KEY"):
    return None
_langfuse_handler = LangfuseCallbackHandler()
return _langfuse_handler
```

Under concurrent first calls this can:

- Let thread B see `_langfuse_checked == True` while `_langfuse_handler` is still `None`, so B skips tracing for its call.
- Let two threads race and each construct a `LangfuseCallbackHandler`, only one of which ends up in `_langfuse_handler` (flaky observability).

There is no lock or `threading.local` around this state, and the tests in `tests/test_llm.py::TestLangfuseHandler` call it sequentially only — they don't exercise the race.

Also note `invoke_model` reads `os.environ.items()` each call to filter `CLAUDECODE`; that's read-only and fine, but the environment is process-global, so mutating it from other threads is not coordinated.

**Bottom line:** `invoke_model` is safe to call from multiple threads for cost tracking and subprocess/API dispatch, but `_get_langfuse_handler` is racy and should be made idempotent (e.g., guard with `_cost_lock` or a separate lock, and only set `_langfuse_checked = True` *after* the handler is built).

## Timeouts — Raised, Not Converted to an `error` Action

`invoke_model` itself **does not implement any `error` action**. It does not catch `subprocess.TimeoutExpired` and does not emit an error dict. The docstring at line ~281 says:

> `Raises subprocess.TimeoutExpired on timeout.`

That's true only for some code paths:

| Model prefix | Where handled | Timeout behavior |
|---|---|---|
| CLI (`claude`, `gemini`, `ollama:*`) | `subprocess.run(..., timeout=timeout)` line ~305 | Raises `subprocess.TimeoutExpired` — **not caught** in `invoke_model`. The `finally:` block still runs (cost/elapsed accounting), then it propagates. |
| `api:*`, `vertex:*` | `_invoke_api` (~line 205) | `llm.invoke` is wrapped; if `"timeout"` appears in the exception *name* or *message*, it is re-raised as `subprocess.TimeoutExpired(model, timeout)`. Otherwise, `RuntimeError(f"{model} failed: {exc}")`. |
| `openai:*` | `_invoke_openai` (~line 258) | Passes `timeout=timeout` into `client.responses.create(...)` but **does not catch** the resulting SDK exception. So a timeout surfaces as the OpenAI SDK's exception type (e.g., `openai.APITimeoutError`), **not** as `subprocess.TimeoutExpired`. |

So the "error action" you're asking about is **not in `reasons/llm.py`**. The observations show:

- `find_error_action_in_llm` returned `total_count: 0` (consistent with there being no literal `"error"` string in `llm.py`).
- `find_invoke_model_callers` shows 124 usages across `reasons/api.py:4022`, `reasons/ask.py:178,328,336,453,506,554`, `reasons/cli.py:1563`, `reasons/build_wiki.py:224`, `reasons/contradictions.py:171,247`, `reasons/pg.py:1142`, `reasons/propose_update.py:323`, `reasons/repair.py:177,199,256,266`, `reasons/repair_premises.py:118`, `reasons/report_belief.py:176`.

Those caller bodies are **not** in the observations, so we cannot see which one wraps `invoke_model` in a `try/except subprocess.TimeoutExpired` and converts it into the specific `error` action the codebase uses. What we *can* say for sure:

1. `invoke_model` raises; it never returns a structured `{"error": ...}` payload.
2. The `error` action — whatever tool/consolidation flow exposes it — is produced by downstream callers, and those callers must handle both `subprocess.TimeoutExpired` (CLI and `api:`/`vertex:`) **and** the SDK's own timeout class (`openai:*`) because the docstring only guarantees the former.
3. The observation `find_timeout_handling` returning `total_count: 0` for pattern `"timeout"` is **inaccurate** — `reasons/llm.py` clearly contains dozens of `timeout` references (`timeout: int = 300`, the arg to `subprocess.run`, `subprocess.TimeoutExpired`, etc.). Treat that observation as unreliable rather than evidence there is no timeout handling.

### What's needed to fully answer the second question

We need to read the try/except blocks around at least:

- `reasons/ask.py:178` (the top-level "ask" path that also passes `timeout=timeout`).
- `reasons/api.py:4009–4022` (the API layer that imports and calls `invoke_model`).
- `reasons/cli.py:1561–1563` (the user-facing CLI entry point).

Without those bodies, the concrete shape of the `error` action (a returned dict, a state transition, a re-raised exception type, an emitted event) cannot be stated.

## Topics to Explore

- [file] `reasons/ask.py` — Contains five separate `invoke_model` call sites (lines 178, 328, 336, 453, 506, 554); inspect them to see whether any catch `subprocess.TimeoutExpired` and translate it into an error action.
- [file] `reasons/api.py` — Imports and calls `invoke_model` around line 4022; likely where the response is wrapped into the outward API contract, including the `error` action.
- [function] `reasons/llm.py:_get_langfuse_handler` — The unsynchronized lazy singleton; confirm the race and decide whether to guard with `_cost_lock` or a dedicated lock.
- [function] `reasons/llm.py:_invoke_openai` — Verify whether the OpenAI SDK timeout exception class is caught upstream by callers, or whether it leaks a different exception type than the docstring promises.
- [general] `error-action-contract` — Locate where the `error` action is defined (probably in the ask/consolidate path) and confirm which exception types it recognizes.

## Beliefs

- `invoke-model-cost-tracking-is-thread-safe` — All mutations of `_cost_tracker` and its nested `by_model` dict go through `_cost_lock`, so concurrent calls do not corrupt cost counters.
- `langfuse-handler-init-is-not-thread-safe` — `_get_langfuse_handler` sets `_langfuse_checked = True` before constructing the handler, so concurrent first calls can observe an incomplete state or build duplicate handlers.
- `invoke-model-raises-timeouts-does-not-return-error` — `invoke_model` propagates `subprocess.TimeoutExpired` (CLI and `api:`/`vertex:` paths) and never returns a structured error value; any `error` action must be produced by its callers.
- `openai-path-timeout-exception-type-differs` — The `openai:*` branch (`_invoke_openai`) does not normalize SDK timeout exceptions to `subprocess.TimeoutExpired`, so callers must handle two different timeout types.
- `invoke-model-docstring-overstates-timeout-guarantee` — The docstring's "Raises subprocess.TimeoutExpired on timeout" is accurate for CLI and `api:`/`vertex:` but not for `openai:`.
