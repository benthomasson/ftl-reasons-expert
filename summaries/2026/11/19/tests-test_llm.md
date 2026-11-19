---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_llm.py

# `tests/test_llm.py` — Test Suite for the Shared LLM Invocation Module

## Purpose

This file is the unit-test suite for **`reasons/llm.py`** (mirrored at `reasons_lib/llm.py`). That module is the project's single abstraction for invoking *any* supported LLM backend through one function — `invoke_model(prompt, model, timeout)`. The tests exist because `llm.py` glues together five heterogeneous transports (Claude CLI, Gemini CLI, Ollama, Anthropic API, Vertex AI) and a process-global cost tracker, and it must pick and shape the right backend purely from a `model` string prefix (e.g. `"claude"`, `"gemini:flash"`, `"ollama:gemma3:4b"`, `"api:claude-sonnet-4-20250514"`, `"vertex:..."`).

The file does **not** test real network or subprocess behavior — every backend is mocked. Its job is to pin the *contract* between model strings, CLI argv, JSON shapes, cost arithmetic, and error types.

## Key Components

The tests are grouped into eight classes, each targeting a slice of `llm.py`:

### `TestResolveModelCmd`
Exercises `resolve_model_cmd(model) -> list[str]`, which only handles **CLI binaries**:
- `"claude"` → `["claude", "-p", "--output-format", "json"]`
- `"claude:sonnet"` → injects `--model sonnet`
- `"gemini"` → `["gemini", "--skip-trust", "-o", "json", "-p", ""]`
- `"gemini:<submodel>"` → injects `-m <submodel>`
- `"ollama:<tag>"` → `["ollama", "run", "<tag>"]`
- Anything else → `ValueError("Unknown model")`

### `TestInvokeModel`
Exercises `invoke_model(prompt, model, timeout=300)`, the top-level dispatcher for CLI backends:
- Missing binary → `FileNotFoundError("... not found in PATH")`
- Success → runs `subprocess.run(cmd, input=prompt, timeout=..., env=..., capture_output=True)`
- Non-zero exit → `RuntimeError("<binary> failed: ...")`
- Ollama output is post-processed: if it begins with `Thinking...` **and** contains `...done thinking.`, everything up to and including the end marker is stripped
- Claude output is **never** thinking-stripped
- The `CLAUDECODE` env var is removed before spawning a child `claude` process (to avoid recursive-agent detection), while `HOME` etc. are preserved

### `TestResolveModelCmdApiModels` / `TestInvokeModelApiDispatch`
These enforce the **prefix partition**:
- `resolve_model_cmd` must *reject* `api:` and `vertex:` with a message that lists both as valid alternatives
- `invoke_model` must route `api:`/`vertex:` to `_invoke_api(prompt, model, timeout)` — passing the full prefixed string and the timeout (default `300`)

### `TestInvokeApi`
Exercises `_invoke_api(prompt, model, timeout)`, the LangChain path:
- `api:claude-sonnet-4-20250514` → `ChatAnthropic(model="claude-sonnet-4-20250514", timeout=120.0)`
- `vertex:...` → `ChatAnthropicVertex(model_name=..., project=..., location=..., request_timeout=...)`
- Vertex `location` falls back to `"us-east5"` when `GOOGLE_CLOUD_REGION` isn't set
- Missing langchain packages → `ImportError` naming the missing package
- Missing `GOOGLE_CLOUD_PROJECT` → `RuntimeError`
- Timeout-looking exceptions → re-raised as `subprocess.TimeoutExpired`
- Any other `invoke()` failure → `RuntimeError("<model> failed: ...")`
- Unknown prefix like `openai:` → `ValueError("Unknown API prefix")`

### `TestLangfuseHandler`
Exercises `_get_langfuse_handler()`, a **lazy singleton**:
- Requires `LANGFUSE_PUBLIC_KEY` in env **and** `_HAS_LANGFUSE`
- Returns the same instance on repeated calls (`LangfuseCallbackHandler` constructed once)
- Returns `None` when either condition is unmet
- The handler is threaded into `invoke()` via `config={"callbacks": [handler]}`; when absent, `config={}`

### `TestParseCliJson`
Exercises `_parse_cli_json(raw_stdout, model) -> str`, the CLI output decoder + cost sink:
- Claude shape: reads `result`, `usage.input_tokens`, `usage.output_tokens`, `total_cost_usd`; **cache** tokens (`cache_creation_input_tokens` + `cache_read_input_tokens`) are added into `input_tokens` (50 + 200 + 100 = 350)
- Gemini shape: reads `response` and `stats.models[*].tokens.{input,candidates}`; USD reported as `0.0`
- Non-JSON / non-object / missing-`result` / top-level-array payloads are all returned **as the raw string**, and **no** cost is recorded (`calls == 0`)

### `TestCostTracker`
Exercises `reset_cost_tracker()`, `_record_cost(model, in_tok, out_tok, usd)`, `get_cost_summary()`, and `format_cost_summary()`:
- Global accumulators: `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `by_model` dict
- `format_cost_summary()` returns `""` when no calls; otherwise formats `$0.0123`, `1,000 input`, `500 output`, `1 call(s)`; omits `$` when cost is `0.0`

## Patterns

- **Mock-at-the-seam**: `subprocess.run` and `shutil.which` are patched from `reasons.llm`, not `subprocess`, so the module's bound reference is what's replaced.
- **Ad-hoc result objects**: `type("Result", (), {"returncode": ..., "stdout": ..., "stderr": ""})()` — a lightweight duck-typed `CompletedProcess` substitute. `completed_process = namedtuple` isn't used; simplicity wins here.
- **`patch.object(llm_module, ...)`** for module globals (`_HAS_LANGCHAIN_ANTHROPIC`, `_HAS_LANGFUSE`, `ChatAnthropic`, `_langfuse_checked`, `_langfuse_handler`). This is essential because `llm.py` caches importability at import time.
- **`patch.dict("os.environ", ...)`** with `clear=True`/`clear=False` variants to isolate env-dependent code paths.
- **`setup_method` / `teardown_method`** in `TestLangfuseHandler` explicitly resets `_langfuse_handler` and `_langfuse_checked` — the singleton cache must not leak between tests. Cost tests use their own `setup_method` calling `reset_cost_tracker()`.
- **Prefix-partition testing**: the same invalid input (`"gpt-4"`) is fed to `resolve_model_cmd` and `_invoke_api` to confirm both error messages advertise the API prefixes.

## Dependencies

**Imported by the test**: `json`, `os`, `subprocess`, `unittest.mock.{MagicMock, patch}`, `pytest`, plus ten symbols from `reasons.llm` (`_get_langfuse_handler`, `_invoke_api`, `_parse_cli_json`, `_record_cost`, `format_cost_summary`, `get_cost_summary`, `invoke_model`, `reset_cost_tracker`, `resolve_model_cmd`) and the module alias `reasons.llm as llm_module`.

**What depends on the tested module**: essentially every other `reasons/*.py` file that calls an LLM — `ask.py`, `derive.py`, `contradictions.py`, `review*.py`, `build_wiki.py`, `cluster.py`, `compact.py`, `import_agent.py`, etc. Changes in `reasons/llm.py` (e.g. a new backend prefix) ripple everywhere, which is why this suite pixens the public surface tightly.

## Flow

The central intended execution path in `llm.py` exercised here:

```
invoke_model(prompt, model="<prefix>:<name>", timeout=300)
  ├─ if model.startswith("api:") or "vertex:"
  │     └─ _invoke_api(prompt, model, timeout)
  │            ├─ get langfuse handler (lazy, cached)
  │            ├─ select ChatAnthropic / ChatAnthropicVertex
  │            └─ model.invoke(prompt, config={"callbacks":[handler]} | {})
  └─ else
        cmd = resolve_model_cmd(model)          # raises ValueError on unknown
        path = shutil.which(cmd[0])             # raises FileNotFoundError if None
        if model.startswith("ollama:"):
              subprocess.run(cmd, input=prompt, ...) → strip thinking → return
        else:
              subprocess.run(cmd, input=prompt, ..., env=sanitized)
                └─ _parse_cli_json(stdout, model) → text + _record_cost(...)
```

`_parse_cli_json` is the only place `_record_cost` is called — so cost accounting is a **side-effect of successful JSON parsing**, not of subprocess success. That's why invalid JSON leaves `calls == 0`.

## Invariants

- **`resolve_model_cmd` handles only CLI backends.** Its `ValueError` message *must* mention `api:<model>` and `vertex:<model>` so callers discover the API path.
- **`invoke_model` default timeout is `300`**, forwarded verbatim to `_invoke_api`; `_invoke_api` converts to `float` when building the LangChain client.
- **Vertex location default is `"us-east5"`**, overridden by `GOOGLE_CLOUD_REGION`.
- **`ChatAnthropic` uses `model=` and `timeout=`, `ChatAnthropicVertex` uses `model_name=` and `request_timeout=`** — the two client APIs differ, and the tests pin this.
- **Ollama thinking stripping requires both `Thinking...` (start) and `...done thinking.` (end)** — a truncated stream is returned untouched.
- **Cost tracker is process-global** and must be reset via `reset_cost_tracker()` between accumulation scenarios.
- **Langfuse handler is a per-process singleton gated by `LANGFUSE_PUBLIC_KEY`**; `_langfuse_checked` acts as a "have we tried yet?" flag.
- **`CLAUDECODE` must not be present in the child process env** when invoking the `claude` CLI.

## Error Handling

The contracts the tests lock in:

| Condition | Raised |
|---|---|
| Unknown non-API prefix in `resolve_model_cmd` / `_invoke_api` | `ValueError` |
| CLI binary not on `PATH` | `FileNotFoundError("... not found in PATH")` |
| CLI exit code != 0 | `RuntimeError("<binary> failed: ...")` |
| Missing `langchain-anthropic` or `langchain-google-vertexai` | `ImportError` naming the package |
| Missing `GOOGLE_CLOUD_PROJECT` for vertex | `RuntimeError` |
| Exception containing "timeout" from API invoke | converted to `subprocess.TimeoutExpired` |
| Any other API invoke failure | `RuntimeError("<model> failed: ...")` |
| Malformed/non-conforming CLI JSON | **swallowed** — raw string returned, no cost recorded |
| Langfuse dep missing or env absent | **swallowed** — returns `None`, `config={}` |

The last two rows are the important "graceful degradation" choices: bad CLI output never breaks a caller (it just returns the string), and missing observability never breaks a call.

## Topics to Explore

- [file] `reasons/llm.py` — The implementation under test; read alongside this suite to see the exact dispatch branches and cost-sink placement.
- [function] `reasons/llm.py:invoke_model` — Entry point that every reasoning command calls; the timeout default and prefix routing live here.
- [function] `reasons/llm.py:_parse_cli_json` — The only caller of `_record_cost`; understanding its tolerance for malformed payloads matters when debugging missing cost data.
- [general] `cost-tracking` — How the process-global `get_cost_summary()` is consumed and rendered elsewhere (e.g. CLI output) and why it must be reset between invocations.
- [general] `langfuse-tracing` — How `_get_langfuse_handler()` feeds callbacks into LangChain `invoke()` and what env vars must be set for traces to appear.

## Beliefs

- `invoke-model-routes-api-prefixes-to-invoke-api` — `invoke_model` dispatches any `api:` or `vertex:` model string directly to `_invoke_api`, bypassing `resolve_model_cmd`.
- `resolve-model-cmd-rejects-api-prefixes` — `resolve_model_cmd` raises `ValueError("Unknown model")` for `api:` and `vertex:` models, and the error text lists both prefixes.
- `cost-tracking-requires-reset-between-scenarios` — `_record_cost` accumulates into process-global state that only `reset_cost_tracker()` clears, so tests must reset between assertions.
- `parse-cli-json-is-the-only-cost-sink` — `_record_cost` is invoked from `_parse_cli_json`; invalid JSON therefore leaves `get_cost_summary()["calls"] == 0`.
- `ollama-thinking-stripping-requires-both-markers` — output lacking either `Thinking...` or `...done thinking.` is returned unchanged, and Claude output is never stripped.
