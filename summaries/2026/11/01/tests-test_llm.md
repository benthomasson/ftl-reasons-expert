---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_llm.py

# `tests/test_llm.py` — Tests for the Shared LLM Invocation Module

## 1. Purpose

This file is the test suite for `reasons/llm.py`, the single module that the entire `ftl-reasons` project uses to talk to LLMs. Its job is to lock down the **abstraction layer** that sits between the application logic (belief derivation, contradiction detection, wiki building, etc.) and the actual model providers.

The responsibility it owns in the test world:

- **Backend dispatch correctness** — that a string like `"claude:sonnet"` or `"ollama:gemma3:4b"` or `"api:claude-sonnet-4-20250514"` resolves to the right invocation path (subprocess vs. langchain client).
- **Output normalization** — that regardless of which CLI tool or API is used, `invoke_model` always returns a clean string.
- **Cost accounting** — that token/cost data is parsed from heterogeneous JSON schemas and accumulated correctly.
- **Optional dependency gating** — that missing optional packages (langchain-anthropic, langchain-google-vertexai, langfuse) degrade gracefully with informative errors rather than import-time crashes.

Every other module in the project (`derive.py`, `contradictions.py`, `build_wiki.py`, etc.) calls into `reasons.llm`, so this test file is the regression net for the entire LLM integration surface.

## 2. Key Components

### Test Classes

| Class | What it pins down |
|---|---|
| `TestResolveModelCmd` | The `resolve_model_cmd()` pure function: string → CLI argv list. Covers claude, gemini (with and without submodel), ollama, and the error path for unknown prefixes. |
| `TestInvokeModel` | The `invoke_model()` orchestration: binary discovery via `shutil.which`, subprocess execution, stdout parsing, Ollama "thinking" stripping, and `CLAUDECODE` env-var sanitization. |
| `TestResolveModelCmdApiModels` | Negative tests: `api:` and `vertex:` prefixes must **not** be handled by `resolve_model_cmd` (they belong to `_invoke_api`). Also asserts the error message lists the valid API prefixes. |
| `TestInvokeModelApiDispatch` | Confirms `invoke_model` routes `api:*` and `vertex:*` to `_invoke_api` and leaves `claude` on the subprocess path. |
| `TestInvokeApi` | The `_invoke_api()` function: langchain `ChatAnthropic` / `ChatAnthropicVertex` construction, timeout→`TimeoutExpired` translation, generic error→`RuntimeError` translation, missing-dependency and missing-env-var errors. |
| `TestLangfuseHandler` | The `_get_langfuse_handler()` singleton: created once when `LANGFUSE_PUBLIC_KEY` is set, `None` otherwise, and threaded through as a langchain callback. |
| `TestParseCliJson` | The `_parse_cli_json()` parser: handles claude's `{"result", "usage", "total_cost_usd"}` schema, gemini's `{"response", "stats.models.*.tokens"}` schema, cache tokens, and falls back to raw text on malformed input. |
| `TestCostTracker` | The module-level cost accumulator: `reset_cost_tracker`, `_record_cost`, `get_cost_summary`, `format_cost_summary`. |

### Key Functions Under Test (from `reasons.llm`)

- **`resolve_model_cmd(model_str) -> list[str]`** — Pure. Maps a human-friendly model string to a CLI argv. No I/O.
- **`invoke_model(prompt, model, timeout=300) -> str`** — Entry point. Dispatches to subprocess or `_invoke_api` based on prefix. Always returns a string or raises.
- **`_invoke_api(prompt, model, timeout) -> str`** — Langchain-based path. Constructs `ChatAnthropic` or `ChatAnthropicVertex`, calls `.invoke()`, returns `.content`.
- **`_parse_cli_json(stdout, model) -> str`** — Extracts the text answer and records cost from provider-specific JSON.
- **`_get_langfuse_handler() -> Optional[object]`** — Lazy singleton for optional tracing.
- **`_record_cost / get_cost_summary / reset_cost_tracker / format_cost_summary`** — Module-level cost accounting.

## 3. Patterns

- **Mock-at-the-boundary.** Every test that touches I/O patches `shutil.which`, `subprocess.run`, or the langchain classes at the module level (`patch.object(llm_module, ...)`). No test ever hits a real network or spawns a real process. The mock boundary is always the OS/executable layer, never internal helpers.

- **Singleton with setup/teardown reset.** `TestLangfuseHandler` and `TestCostTracker` both call `llm_module._langfuse_handler = None` / `reset_cost_tracker()` in `setup_method` to destroy module-level state. This is the standard pattern for testing module-level mutable state without a class instance.

- **Duck-typed mock for `subprocess.run` result.** Instead of using `MagicMock` for the subprocess result, the tests use a tiny local class:
  ```python
  mock_result = type("Result", (), {"returncode": 0, "stdout": json_out, "stderr": ""})()
  ```
  This is a deliberate choice — it makes the shape of the expected result object explicit and avoids `MagicMock`'s silent attribute creation, which could mask a typo in an attribute name the code reads.

- **Parameterized negative tests for error messages.** `TestResolveModelCmdApiModels::test_error_message_lists_api_models` asserts that the `ValueError` message contains `"api:<model>"` and `"vertex:<model>"`. This locks in the user-facing error contract.

- **Environment isolation via `patch.dict("os.environ", ...)`.** Used for `CLAUDECODE`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_REGION`, `LANGFUSE_PUBLIC_KEY`, `HOME`. `TestInvokeApi::test_vertex_default_region` goes further and uses `clear=True` to guarantee the region is absent, testing the default-fallback path.

- **Accumulation vs. reset lifecycle.** `TestCostTracker` and `TestParseCliJson` both call `reset_cost_tracker()` at the start of each test (via `setup_method` or inline) to prevent cross-test contamination from the module-level accumulator.

## 4. Dependencies

### This file depends on

| Dependency | Role |
|---|---|
| `reasons.llm` (the module under test) | All public and private functions listed above. |
| `unittest.mock.MagicMock`, `patch` | I/O mocking. |
| `pytest` | Test runner, `pytest.raises`, `pytest.approx`. |
| `json`, `os`, `subprocess` | Used in mock data construction and asserting on `subprocess.TimeoutExpired`. |

### What depends on this file (indirectly)

Everything that imports `reasons.llm` is transitively covered by these tests:

- `reasons/derive.py`, `reasons/contradictions.py`, `reasons/build_wiki.py`, `reasons/repair.py`, `reasons/propose_update.py`, `reasons/review.py`, `reasons/ask.py`, `reasons/compact.py`, `reasons/summarize` (via `export_markdown.py`), and others.

There is no `imported_by` graph within the test suite itself — this is a leaf test file.

## 5. Flow

The tests exercise two major execution paths through `invoke_model`:

### Path A: CLI subprocess (claude, gemini, ollama)

```
invoke_model("hello", model="claude")
  │
  ├─ model has no "api:"/"vertex:" prefix
  │
  ├─ resolve_model_cmd("claude")
  │    → ["claude", "-p", "--output-format", "json"]
  │
  ├─ shutil.which("claude") → "/usr/bin/claude"   (mocked)
  │
  ├─ Build env: copy os.environ, delete "CLAUDECODE"
  │
  ├─ subprocess.run(cmd, input="hello", timeout=60, env=cleaned_env)
  │    → Result(returncode=0, stdout='{"result":"response",...}')
  │
  ├─ returncode != 0?  → raise RuntimeError("claude failed: ...")
  │
  ├─ _parse_cli_json(stdout, "claude")
  │    → json.loads → extract "result" field
  │    → _record_cost("claude", input_tokens, output_tokens, cost)
  │    → return "response"
  │
  └─ (ollama only) strip "Thinking... ...done thinking." block
       → return cleaned text
```

### Path B: API via langchain (api:*, vertex:*)

```
invoke_model("hello", model="api:claude-sonnet-4-20250514")
  │
  ├─ model starts with "api:" or "vertex:"
  │
  └─ _invoke_api("hello", "api:claude-sonnet-4-20250514", timeout=300)
       │
       ├─ Strip prefix → "claude-sonnet-4-20250514"
       │
       ├─ api: → ChatAnthropic(model=..., timeout=...)
       │  vertex: → ChatAnthropicVertex(model_name=..., project=..., location=...)
       │
       ├─ _get_langfuse_handler() → handler or None
       │
       ├─ model.invoke("hello", config={"callbacks": [handler]})
       │    → response.content
       │
       ├─ Exception "timed out" → raise subprocess.TimeoutExpired
       ├─ Other Exception → raise RuntimeError("api:... failed: ...")
       │
       └─ return "test response"
```

### Cost tracking (cross-cutting)

Every successful call funnels through `_record_cost()`, which increments a module-level dict. `get_cost_summary()` reads it; `format_cost_summary()` renders a human-readable string; `reset_cost_tracker()` zeroes it. Tests verify accumulation, per-model breakdown, and the empty-state format.

## 6. Invariants

1. **`invoke_model` always returns a `str` or raises.** It never returns `None`, a list, or a raw dict. The tests in `TestInvokeModel` and `TestInvokeModelApiDispatch` assert `result == "some string"` for every success path.

2. **`resolve_model_cmd` rejects `api:` and `vertex:` prefixes.** These are reserved for `_invoke_api`. The negative tests in `TestResolveModelCmdApiModels` enforce that mixing the two dispatch mechanisms raises `ValueError`.

3. **Ollama "thinking" stripping is conditional.** The block is only removed when it contains **both** a `Thinking...` start marker and a `...done thinking.` end marker. `test_ollama_incomplete_thinking_unchanged` locks in that a partial thinking block is left intact.

4. **Claude output is never post-processed for thinking.** `test_claude_does_not_strip_thinking` asserts the raw `result` field passes through unchanged, even if it contains thinking-like text. The stripping is Ollama-specific.

5. **The Langfuse handler is a singleton.** `_get_langfuse_handler()` creates the handler at most once per process. `TestLangfuseHandler::test_handler_singleton` asserts `MockCB.assert_called_once()` across two invocations.

6. **`CLAUDECODE` env var is always stripped** before spawning the claude subprocess. This prevents the claude CLI from detecting it's running inside another claude session and changing behavior.

7. **Cost tracker is reset before each test** that touches it, preventing order-dependent failures.

## 7. Error Handling

The tests encode a precise error taxonomy:

| Condition | Exception | Trigger |
|---|---|---|
| Binary not on PATH | `FileNotFoundError` with `"not found in PATH"` | `shutil.which` returns `None` |
| Subprocess exits non-zero | `RuntimeError` with `"<model> failed"` | `returncode != 0` |
| Subprocess times out | `subprocess.TimeoutExpired` | `subprocess.run` raises it |
| API model times out | `subprocess.TimeoutExpired` (re-raised) | `_invoke_api` catches `Exception` with "timed out" in message and converts |
| API auth / other failure | `RuntimeError` with `"<model> failed"` | `_invoke_api` catches non-timeout `Exception` |
| Unknown model prefix in `resolve_model_cmd` | `ValueError` with `"Unknown model"` and a hint listing `api:<model>` / `vertex:<model>` | Unrecognized first token |
| `api:`/`vertex:` passed to `resolve_model_cmd` | `ValueError` | These prefixes are not CLI tools |
| `openai:` or other unknown API prefix in `_invoke_api` | `ValueError` with `"Unknown API prefix"` | Not `api:` or `vertex:` |
| `langchain-anthropic` not installed | `ImportError` with `"langchain-anthropic"` | `_HAS_LANGCHAIN_ANTHROPIC` is `False` |
| `langchain-google-vertexai` not installed | `ImportError` with `"langchain-google-vertexai"` | `_HAS_LANGCHAIN_VERTEX` is `False` |
| `GOOGLE_CLOUD_PROJECT` not set for vertex | `RuntimeError` with `"GOOGLE_CLOUD_PROJECT"` | Missing env var |

The notable design choice: **API timeouts are re-raised as `subprocess.TimeoutExpired`** rather than a custom exception. This lets callers (like `derive.py`) use a single `except subprocess.TimeoutExpired` clause regardless of whether the call went through subprocess or langchain. The test `test_timeout_reraises_as_timeout_expired` explicitly guards this contract.

Malformed JSON in `_parse_cli_json` is **not** an error — it falls through to returning the raw string. The tests `test_invalid_json_returns_raw`, `test_json_missing_result_returns_raw`, and `test_json_array_returns_raw` confirm this graceful degradation, and verify that no cost is recorded in these cases.

---

## Topics to Explore

- [file] `reasons/llm.py` — The module under test. Read the actual implementation of `invoke_model`, `_invoke_api`, `_parse_cli_json`, and the cost tracker to understand how the mocks map to real code, especially the Ollama thinking-stripping regex and the `_HAS_*` import guards.
- [file] `reasons/derive.py` — The highest-volume consumer of `invoke_model`. See how it handles `TimeoutExpired` vs `RuntimeError` and how it threads the model string from CLI/config into the LLM call.
- [function] `reasons/llm.py:_parse_cli_json` — The provider-specific JSON parsing. Understanding the claude vs. gemini schema differences and the cache-token aggregation logic is essential before modifying either provider's output format.
- [file] `tests/conftest.py` — Check for shared fixtures (e.g., a mock `invoke_model` fixture) that other test files use. This file does **not** use a fixture for the subprocess mock — it builds inline mocks — which is a style choice worth comparing against the rest of the suite.
- [general] `optional-dependency-gating` — The `_HAS_LANGCHAIN_ANTHROPIC`, `_HAS_LANGCHAIN_VERTEX`, `_HAS_LANGFUSE` pattern. Explore how `reasons/llm.py` sets these at import time and whether other modules follow the same pattern for their optional deps.

---

## Beliefs

- `invoke-model-normalizes-to-str` — `invoke_model()` always returns a plain `str` on success and raises on failure; it never returns a dict, `None`, or a langchain response object.
- `timeout-semantics-unified` — Both the subprocess path and the langchain API path surface timeouts as `subprocess.TimeoutExpired`, allowing callers to use a single exception handler regardless of backend.
- `ollama-thinking-strip-is-bounded` — Ollama output stripping only removes text between `Thinking...` and `...done thinking.` markers; a missing end marker leaves the output untouched.
- `resolve-model-cmd-rejects-api-prefixes` — `resolve_model_cmd()` raises `ValueError` for any string starting with `api:` or `vertex:`, enforcing that those prefixes are exclusively handled by `_invoke_api()`.
- `langfuse-handler-is-process-singleton` — `_get_langfuse_handler()` instantiates the `LangfuseCallbackHandler` at most once per process lifetime; subsequent calls return the cached instance without re-reading env vars.
