---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `_get_langfuse_handler()` feeds callbacks into LangChain `invoke()` and what env vars must be set for traces to appear.

# How `_get_langfuse_handler()` Feeds Callbacks into LangChain `invoke()`

## 1. The handler factory (`reasons/llm.py:157-167`)

```python
def _get_langfuse_handler():
    global _langfuse_handler, _langfuse_checked
    if _langfuse_checked:
        return _langfuse_handler
    _langfuse_checked = True
    if not _HAS_LANGFUSE:
        return None
    if not os.environ.get("LANGFUSE_PUBLIC_KEY"):
        return None
    _langfuse_handler = LangfuseCallbackHandler()
    return _langfuse_handler
```

Three things to note:

1. **Package-availability gate** — `_HAS_LANGFUSE` is set at import time (lines 35–39):
   ```python
   try:
       from langfuse.langchain import CallbackHandler as LangfuseCallbackHandler
       _HAS_LANGFUSE = True
   except ImportError:
       LangfuseCallbackHandler = None
       _HAS_LANGFUSE = False
   ```
   So tracing is silently disabled unless the `langfuse` package (and its `langchain` integration submodule) is importable. In this repo that comes from the optional extra `langfuse = ["langfuse>=2.0"]` in `pyproject.toml`, or the bundled `llm` extra.

2. **Env-var gate** — only `LANGFUSE_PUBLIC_KEY` is checked explicitly on line 164. If unset (or empty), the function returns `None` and no callback is ever attached.

3. **Process-wide caching** — `_langfuse_checked` flips to `True` on the first call, and `_langfuse_handler` is memoized in module globals. This is verified in `tests/test_llm.py:311-315`, which asserts `first is second` (returned object is a singleton) and in `tests/test_llm.py:298-306`, which asserts a `None` result under three different failure conditions: import missing, env-var missing, or pre-set negative cache.

   The practical consequence: if the first invocation happens before `LANGFUSE_PUBLIC_KEY` is exported, later calls in the *same process* will keep returning `None`. You must set the key before the process touches an API-model invocation.

## 2. How it becomes a LangChain callback (`reasons/llm.py:234-239`)

Inside the API-provider path (the function is clipped at line 200 in the observations, but the grep for `callbacks` and `invoke` pins it down):

```python
handler = _get_langfuse_handler()        # line 234
if handler is not None:                  # line 235 (implied)
    config["callbacks"] = [handler]      # line 236
response = llm.invoke(prompt, config=config)  # line 239
```

So the flow is:

- `handler is None` → `config["callbacks"]` is not set → `llm.invoke()` runs without tracing.
- `handler is not None` → a one-element list `[handler]` is placed in the `callbacks` field of the LangChain `Runnable` config → `llm.invoke(prompt, config=config)` runs the `LangfuseCallbackHandler` for that specific call.

The plumb-through is directly exercised by `tests/test_llm.py:319-334` (`test_langfuse_callback_passed_to_invoke`), which mocks `invoke`, then asserts `call_config["callbacks"] == [mock_handler]` on line 334. That test is the authoritative proof that `invoke()` receives the handler in its `config` dict — nothing else in the module wires the callback.

Note the scope: the handler is only attached in the **API model branch** (`api:` / `vertex:` / `openai:` paths). CLI models (`claude`, `gemini`, `ollama:`) go through `subprocess.Popen`-style dispatch via `resolve_model_cmd()` and `_parse_cli_json()`, bypassing `_invoke_api` entirely — so CLI calls are **not** traced.

## 3. What env vars are required for traces to appear

Two independent gates must both open:

| Gate | Requirement | Source |
|---|---|---|
| Import | `langfuse` (with `langfuse.langchain`) importable in the active virtualenv | `reasons/llm.py:35-39`; `pyproject.toml` `langfuse`/`llm` extras |
| Runtime | `LANGFUSE_PUBLIC_KEY` set to a non-empty string in `os.environ` **before the first `_get_langfuse_handler()` call in the process** | `reasons/llm.py:164` |

Beyond those two repo-local checks, the Langfuse SDK itself needs enough credentials/endpoint to actually publish a trace. The observations do **not** show the SDK internals, so from this codebase alone we can only see `LANGFUSE_PUBLIC_KEY` being validated — but in practice Langfuse's `CallbackHandler()` also reads `LANGFUSE_SECRET_KEY` (and optionally `LANGFUSE_HOST` / `LANGFUSE_BASE_URL` for self-hosted) internally. If either of those isn't set, the handler still gets attached and `invoke()` still works, but the SDK will fail to authenticate against the Langfuse API and no traces reach the server.

Test fixtures reflect this: the tests only stub `LANGFUSE_PUBLIC_KEY = "pk-test"` (e.g. `tests/test_llm.py:293`) while patching `_HAS_LANGFUSE = True` and *mocking* `LangfuseCallbackHandler`, so they never exercise the real credential path.

### Quick checklist to make traces appear

1. `pip install -e ".[langfuse]"` (or `.[llm]`) so `from langfuse.langchain import CallbackHandler` succeeds → `_HAS_LANGFUSE = True`.
2. Export **before** launching the process:
   ```bash
   export LANGFUSE_PUBLIC_KEY="pk-lf-..."
   export LANGFUSE_SECRET_KEY="sk-lf-..."
   export LANGFUSE_HOST="https://cloud.langfuse.com"   # or your self-hosted URL
   ```
3. Use an API-style model name (`api:...`, `vertex:...`, `openai:...`) — CLI models won't emit traces.
4. Confirm `_get_langfuse_handler()` returns a handler (not `None`) at first call — once it returns `None`, it's cached for the rest of the process (`_langfuse_checked` never resets).

## Topics to Explore

- [function] `reasons/llm.py:_invoke_api` — The full body of the function that assembles `config`, checks `handler is not None`, and calls `llm.invoke(prompt, config=config)`; the observations cut off at line 200 so the branches for `api:`, `vertex:`, and `openai:` are only inferred.
- [file] `tests/test_llm.py` — Lines ~290–340 are the canonical spec for the caching, gate, and callback-attachment behavior; useful as a regression harness to extend when changing handler wiring.
- [file] `pyproject.toml` — Shows the three optional-extras tiers (`langfuse`, `api`, `llm`) that control whether `_HAS_LANGFUSE`/`_HAS_LANGCHAIN_ANTHROPIC` become `True`.
- [general] `langfuse-sdk-env-vars` — The Langfuse SDK's own env vars (`LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`/`LANGFUSE_BASE_URL`, sampling, etc.) are not visible in this repo; check the Langfuse docs to know which values are actually required for tracing versus just passing the local gate.
- [general] `langfuse-v2-vs-v3-import-path` — The import is `from langfuse.langchain import CallbackHandler`, which is the Langfuse v3 module layout; `pyproject.toml` pins `langfuse>=2.0` with no upper bound, so verify the installed major version still exposes `langfuse.langchain`.

## Beliefs

- `langfuse-handler-requires-public-key` — `_get_langfuse_handler()` returns `None` whenever `LANGFUSE_PUBLIC_KEY` is missing or empty, regardless of any other Langfuse env var.
- `langfuse-handler-is-process-singleton` — The handler (or the `None` result) is cached in module-level `_langfuse_handler` / `_langfuse_checked` globals, so the outcome is fixed by the first call in each process.
- `langfuse-enables-invoke-tracing` — Tracing only occurs when `config["callbacks"] = [handler]` is set on the `config` dict passed to `llm.invoke(...)` at `reasons/llm.py:236-239`.
- `langfuse-handler-requires-langfuse-import` — `_HAS_LANGFUSE` is `True` only if `from langfuse.langchain import CallbackHandler` succeeds at module import time; otherwise the factory short-circuits to `None` even with the env var set.
- `langfuse-tracing-is-api-path-only` — CLI-backed models dispatched via `resolve_model_cmd()` never invoke `_get_langfuse_handler()` and therefore produce no Langfuse traces.
