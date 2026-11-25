---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The Langfuse SDK's own env vars (`LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`/`LANGFUSE_BASE_URL`, sampling, etc.) are not visible in this repo; check the Langfuse docs to know which values are actually required for tracing versus just passing the local gate.

## What this repo actually enforces

The only Langfuse-related configuration logic in this repo is a local gate in `_get_langfuse_handler` at `reasons/llm.py:157-167`:

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

The gate has three conditions:

1. `_HAS_LANGFUSE` — the `from langfuse.langchain import CallbackHandler as LangfuseCallbackHandler` import at `reasons/llm.py:26-31` succeeded.
2. `LANGFUSE_PUBLIC_KEY` is set in the environment.
3. If both hold, a bare `LangfuseCallbackHandler()` is constructed with **no arguments**.

That is the entirety of the configuration surface the repo exposes. There is no reference anywhere to `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`, `LANGFUSE_BASE_URL`, sampling env vars, `sample_rate`, or any Langfuse client options — neither in `reasons/llm.py`, nor elsewhere (the all-files `LANGFUSE_` search returns only the four hits above in `reasons/llm.py` and `tests/test_llm.py`).

## What that means in practice

Because the handler is instantiated as `LangfuseCallbackHandler()`, the actual Langfuse client configuration is delegated entirely to the SDK's default env-var resolution. The repo therefore only answers "should I attempt to attach a handler?" — it does not answer "is the handler correctly configured to ship traces?"

Concretely:

- **Passing the gate ≠ tracing works.** Setting `LANGFUSE_PUBLIC_KEY=pk-…` only passes the `os.environ.get(...)` check. Whether the SDK then succeeds in exporting spans depends on the SDK's own requirements at construction / first flush (typically including a secret key and a resolvable host). If those are missing, the failure will be raised or logged *inside the SDK*, not in this repo's code.
- **The gate is intentionally minimal.** It is not a config validator; it is a cheap "is tracing plausibly enabled?" check, so that a missing public key avoids the cost of importing/instantiating the SDK.
- **The handler is memoized.** `_langfuse_checked` is set on the first call, so all required Langfuse env vars must be present before the first API-prefixed or vertex-prefixed call reaches `_get_langfuse_handler`. Setting them after the first call has no effect because of `if _langfuse_checked: return _langfuse_handler`.
- **Only API/vertex paths are traced.** The callback is attached in `_invoke_api` (`reasons/llm.py`, the `config = {}` / `handler = _get_langfuse_handler()` block). The CLI paths (`claude`, `gemini`, `ollama`) run through subprocess and never touch Langfuse.

## Why the repo can't tell you what's *actually* required

Two structural reasons:

1. **No SDK-internal constants are copied into the repo.** The tests in `tests/test_llm.py:293,305,313` only pass `LANGFUSE_PUBLIC_KEY="pk-test"` — i.e. they test the gate, not end-to-end tracing. There is no fixture that stands up a fake Langfuse endpoint, so "which env vars are enough to actually publish a trace" is deliberately out of scope for this codebase's tests.
2. **`langfuse` is an optional, loosely pinned dependency.** `pyproject.toml` declares `langfuse = ["langfuse>=2.0"]` and `llm = [..., "langfuse>=2.0"]`. Since there is no upper bound and no lockfile visible here, the exact set of accepted env var names (`LANGFUSE_HOST` vs `LANGFUSE_BASE_URL`, rate-limit / sampling env vars, `LANGFUSE_TRACING_ENABLED`, etc.) depends on whichever `langfuse` version the user installs. Those names are never referenced in this repo.

So the correct mental model is: **this codebase owns the on/off decision (`LANGFUSE_PUBLIC_KEY` presence), and the Langfuse SDK owns everything else (secret key, host, sampling, flushing).** To know which values are *required* versus merely *sufficient to pass the local gate*, check the Langfuse docs for the installed SDK version (the v2 docs cover `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`/`LANGFUSE_BASE_URL`, and sampling behavior).

## What is missing from the observations

If you need to go beyond this, the following aren't visible in the data above:

- The installed `langfuse` version (e.g. via a lockfile or `pip freeze`), which determines which env var names are recognized and whether the handler raises vs. logs when the secret key is absent.
- Any runtime logs from `LangfuseCallbackHandler()` showing whether construction errored or silently no-op'd when only `LANGFUSE_PUBLIC_KEY` was set.
- The body of the earlier `_invoke_api` config-attachment block was in the excerpt, but the surrounding flow (`invoke_model` dispatch into `_invoke_api`) is only partially shown.

## Topics to Explore

- [function] `reasons/llm.py:_get_langfuse_handler` — The entire Langfuse gate; reading it confirms only `LANGFUSE_PUBLIC_KEY` is checked locally.
- [file] `tests/test_llm.py` — The tests around lines 293–315 only exercise the gate with `pk-test`, not end-to-end trace export.
- [function] `reasons/llm.py:_invoke_api` — Where the callback is attached to `config["callbacks"]`; also the only call path that uses tracing.
- [general] `langfuse-v2-env-vars` — The Langfuse docs page describing `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`/`LANGFUSE_BASE_URL`, and sampling options for the `langfuse>=2.0` version declared in `pyproject.toml`.
- [file] `pyproject.toml` — Confirms `langfuse` is optional with a lower bound but no upper pin, so env-var behavior is version-dependent.

## Beliefs

- `langfuse-gate-checks-only-public-key` — `_get_langfuse_handler` at `reasons/llm.py:164` returns `None` unless `LANGFUSE_PUBLIC_KEY` is set; it does not inspect `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`, `LANGFUSE_BASE_URL`, or sampling settings.
- `langfuse-handler-uses-sdk-default-config` — The callback is constructed as `LangfuseCallbackHandler()` with zero arguments, so all connection and sampling configuration is resolved by the Langfuse SDK from its own env vars.
- `repo-has-no-langfuse-secret-key-reference` — The repo contains no reference to `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`, `LANGFUSE_BASE_URL`, or sampling env vars (the all-files `LANGFUSE_` search returns only four hits, all `LANGFUSE_PUBLIC_KEY`).
- `langfuse-handler-is-cached-on-first-call` — `_langfuse_checked` short-circuits subsequent lookups, so Langfuse env vars must be set before the first `_get_langfuse_handler` call; later changes are ignored.
- `tracing-only-enables-on-api-and-vertex-prefixes` — The `config["callbacks"]` wiring lives inside `_invoke_api`, so the `claude`, `gemini`, and `ollama` CLI paths never use the Langfuse callback.
