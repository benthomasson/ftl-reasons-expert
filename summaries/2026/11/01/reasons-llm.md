---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/llm.py

# `reasons/llm.py` — Shared LLM Invocation Layer

## 1. Purpose

This file is the **single chokepoint for all LLM calls** in the ftl-reasons project. Every module that needs to "ask a model something" (review, verify, repair, derive, contradictions, etc.) funnels through `invoke_model()` here rather than talking to a provider directly.

It owns two responsibilities:

1. **Provider abstraction** — translates a string like `"claude:sonnet"` or `"vertex:claude-haiku"` into an actual invocation (CLI subprocess or HTTP API call) and returns plain response text.
2. **Cost & token accounting** — accumulates input/output token counts, USD cost, call counts, and wall-clock time across the process lifetime, exposed via `get_cost_summary()` / `format_cost_summary()`.

Because it is imported by 15+ modules (see `imported_by`), any change to the model-string grammar or the return contract of `invoke_model()` is a breaking change across the codebase.

## 2. Key Components

### Constants & Module State

| Symbol | Role |
|---|---|
| `MODEL_COMMANDS` | Dict mapping `"claude"` / `"gemini"` to their base CLI arg lists. This is the "no-submodel" fast path. |
| `_cost_tracker` | Module-level dict holding cumulative stats (`calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `elapsed_seconds`, `by_model`). Mutated only under `_cost_lock`. |
| `_cost_lock` | `threading.Lock()` guarding every read/write of `_cost_tracker`. |
| `_langfuse_handler` / `_langfuse_checked` | Lazy-singleton pair for the optional Langfuse callback. Checked once, then cached. |

### Public API (what other modules call)

- **`invoke_model(prompt, model="claude", timeout=300) -> str`**
  The sole public entry point. Returns the model's response as a plain string. Raises on failure (see §7). This is the function every caller in `reasons/` uses.

- **`get_cost_summary() -> dict`**
  Returns a deep copy of the accumulated cost tracker. Safe to call from any thread.

- **`format_cost_summary() -> str`**
  Human-readable one-liner, e.g. `"Cost: $0.0042 | 1,203 input + 456 output tokens | 3 call(s) | 12.3s"`. Returns `""` if no calls have been made.

- **`reset_cost_tracker()`**
  Zeros out all counters. Useful in tests.

### Internal / Private Functions

- **`resolve_model_cmd(model) -> list[str]`**
  Maps a model string to a CLI argv list. Handles the `"claude:<sub>"`, `"gemini:<sub>"`, and `"ollama:<model>"` syntax. Raises `ValueError` for unknown prefixes (including `"openai:"`, which is explicitly rejected here because OpenAI goes through `_invoke_openai`).

- **`_parse_cli_json(output, model) -> str`**
  Parses the `--output-format json` envelope that Claude CLI and Gemini CLI emit. Extracts the response text and records token/cost stats. **Falls back to returning the raw string** if JSON parsing fails — this is a deliberate resilience choice so a provider format change doesn't crash the pipeline.

- **`_invoke_api(prompt, model, timeout) -> str`**
  Handles `"api:<name>"` (Anthropic via `ChatAnthropic`) and `"vertex:<name>"` (via `ChatAnthropicVertex`). Optionally attaches a Langfuse callback. Normalises timeout exceptions into `subprocess.TimeoutExpired` so callers have a single exception type to catch.

- **`_invoke_openai(prompt, model, timeout) -> str`**
  Uses the OpenAI **Responses API** (`client.responses.create`), not the older Chat Completions. Iterates `response.output` items to extract message text. Records tokens but sets cost to `0.0` (the Responses API doesn't return USD cost in the same shape).

- **`_record_cost(model, input_tokens, output_tokens, cost_usd)`**
  Thread-safe increment of the global and per-model counters.

- **`_get_langfuse_handler()`**
  Returns `None` unless both `langfuse` is installed **and** `LANGFUSE_PUBLIC_KEY` is set. Caches the result after first check.

## 3. Patterns

- **Strategy / dispatch by string prefix.** `invoke_model` does a simple `startswith` cascade (`api:`, `vertex:`, `openai:`, then CLI). Adding a new provider means adding one `if` branch and one `_invoke_*` function. There is no class hierarchy or plugin registry — the codebase favours the flat, grep-able approach.

- **Graceful optional imports.** Every third-party import (`langchain_anthropic`, `langchain_google_vertexai`, `langfuse`, `openai`) is wrapped in `try/except ImportError` and sets a `_HAS_*` flag. This lets the package install with zero optional deps and only fail at *call time* if a specific provider is requested. The error message tells the user exactly which pip extra to install.

- **Lazy singleton.** The Langfuse handler is created at most once (`_langfuse_checked` guard) and shared across all API calls.

- **Thread-safe accumulator.** `_cost_lock` is held for the entire read-or-write of `_cost_tracker`. `get_cost_summary` uses `copy.deepcopy` so callers get a stable snapshot.

- **Subprocess with stripped env.** The `CLAUDECODE` environment variable is explicitly removed from the subprocess `env` dict. This prevents the Claude CLI from entering a "nested agent" mode that would alter its output format.

- **Fragile string parsing with a comment.** The Ollama "Thinking..." / "...done thinking." stripping is explicitly marked as version-fragile. If you're maintaining this, expect it to break on Ollama updates.

## 4. Dependencies

### This file imports
| Dependency | Usage |
|---|---|
| `json`, `os`, `shutil`, `subprocess`, `threading` | stdlib — CLI invocation, env inspection, thread safety |
| `langchain_anthropic.ChatAnthropic` *(optional)* | `"api:"` provider |
| `langchain_google_vertexai.ChatAnthropicVertex` *(optional)* | `"vertex:"` provider |
| `langfuse.langchain.CallbackHandler` *(optional)* | Tracing for API calls |
| `openai` *(optional)* | `"openai:"` provider (Responses API) |
| `copy`, `time` | Imported locally inside functions to keep module import lightweight |

### Modules that import this file
`reasons/api.py`, `ask.py`, `build_wiki.py`, `cli.py`, `contradictions.py`, `pg.py`, `propose_update.py`, `repair.py`, `repair_premises.py`, `report_belief.py`, `report_gated.py`, `review.py`, `review_justifications.py`, `review_premises.py`, `verify.py`, plus the mirror `reasons_lib/llm.py` and `tests/test_llm.py`.

> **Note:** There is a parallel `reasons_lib/llm.py`. If you change one, check whether the other needs the same change (or whether they've diverged).

## 5. Flow

```
Caller (e.g. reasons/verify.py)
  │
  ▼
invoke_model(prompt, model="claude:sonnet", timeout=300)
  │
  ├─ model starts with "api:" or "vertex:"?
  │     YES → _invoke_api()
  │           ├─ build LangChain chat object
  │           ├─ attach Langfuse handler (if configured)
  │           ├─ llm.invoke(prompt, config)
  │           └─ return response.content
  │
  ├─ model starts with "openai:"?
  │     YES → _invoke_openai()
  │           ├─ OpenAI().responses.create(...)
  │           ├─ extract text from response.output
  │           ├─ _record_cost(...)
  │           └─ return text
  │
  └─ else (CLI model)
        │
        ├─ resolve_model_cmd("claude:sonnet")
        │   → ["claude","-p","--model","sonnet","--output-format","json"]
        │
        ├─ shutil.which("claude")  ← FileNotFoundError if missing
        │
        ├─ subprocess.run(cmd, input=prompt, capture_output=True,
        │                  timeout=300, env={...no CLAUDECODE...})
        │
        ├─ returncode != 0?  → RuntimeError
        │
        ├─ ollama? → strip "Thinking..." markers → return raw text
        │
        └─ else → _parse_cli_json(stdout, model)
                   ├─ json.loads(stdout)
                   ├─ extract .result (claude) or .response (gemini)
                   ├─ extract usage/stats → _record_cost()
                   └─ return text (or raw stdout on parse failure)
  │
  finally:
    record elapsed_seconds into _cost_tracker (global + per-model)
```

The `finally` block runs on **every** path (including exceptions), so wall-clock time is always accounted.

## 6. Invariants

1. **Model string grammar is closed.** Valid forms are: `claude`, `gemini`, `claude:<sub>`, `gemini:<sub>`, `ollama:<model>`, `api:<model>`, `vertex:<model>`, `openai:<model>`. Anything else raises `ValueError` from `resolve_model_cmd`.

2. **`invoke_model` always returns a `str`** (the model's text response) or raises. It never returns `None`, a dict, or a LangChain object. Callers can safely do `result.strip()` or pass it into a template.

3. **Cost tracking is process-lifetime and thread-safe.** There is no per-call return of cost data; you query `get_cost_summary()` after the fact. The lock ensures no torn reads under concurrent `invoke_model` calls.

4. **`CLAUDECODE` is never present in the subprocess environment.** This is a hard invariant enforced by the dict comprehension on the `env` parameter.

5. **Ollama responses bypass JSON parsing.** Ollama has no `--output-format json` mode in the way Claude/Gemini do, so its output is returned raw (after thinking-marker stripping). No cost is recorded for Ollama calls (the `_record_cost` call is skipped).

## 7. Error Handling

| Condition | Exception | Where raised |
|---|---|---|
| CLI binary not in PATH | `FileNotFoundError` | `invoke_model`, via `shutil.which` check |
| Non-zero exit code from CLI | `RuntimeError(f"{model} failed: {stderr}")` | `invoke_model` |
| Timeout (CLI) | `subprocess.TimeoutExpired` | `subprocess.run` (propagated) |
| Timeout (API) | Normalised to `subprocess.TimeoutExpired` | `_invoke_api` catches and re-raises |
| Missing `langchain-anthropic` | `ImportError` with pip hint | `_invoke_api` |
| Missing `langchain-google-vertexai` | `ImportError` with pip hint | `_invoke_api` |
| Missing `GOOGLE_CLOUD_PROJECT` env var | `RuntimeError` | `_invoke_api` |
| Missing `openai` package | `ImportError` with pip hint | `_invoke_openai` |
| Unknown model prefix | `ValueError` (lists all valid forms) | `resolve_model_cmd` |
| API call failure (non-timeout) | `RuntimeError(f"{model} failed: {exc}")` | `_invoke_api` |
| JSON parse failure (CLI output) | **Not raised** — falls back to raw string | `_parse_cli_json` |

Notably, `_parse_cli_json` **swallows** `json.JSONDecodeError` and returns the raw output. This means a provider that changes its JSON envelope won't crash the pipeline; it will just stop recording costs. That's a deliberate trade-off for resilience, but it means cost tracking can silently go to zero if the format shifts.

---

## Topics to Explore

- [file] `reasons_lib/llm.py` — The parallel copy of this module. Determine whether it is a stale duplicate, a fork with divergent changes, or an intentional mirror. Any bugfix you make here may need to be duplicated there.
- [function] `reasons/llm.py:_parse_cli_json` — The JSON envelope parsing differs between Claude (`data["result"]`, `data["usage"]`) and Gemini (`data["response"]`, `data["stats"]["models"]`). Understand the exact contract of each CLI's `--output-format json` output, because a provider update here silently breaks cost tracking.
- [file] `tests/test_llm.py` — See how the test suite mocks or stubs the subprocess and API layers. This tells you what seams are considered testable and what the team expects to be injectable.
- [general] `ollama-thinking-stripping` — The `"Thinking...\n"` / `"...done thinking.\n"` split is explicitly marked fragile. Check Ollama's changelog for plans to change or standardise reasoning-token output, since this will break silently.
- [file] `reasons/verify.py` — A representative caller. See how `invoke_model`'s return value is consumed, how errors are caught, and whether cost summaries are surfaced to the user, to understand the full data flow from prompt to cost report.

---

## Beliefs

- `invoke-model-returns-plain-str` — `invoke_model()` always returns a `str` (the model's response text) and never returns a structured object; all 15+ caller modules depend on this contract.
- `cli-models-strip-claudecode-env` — The subprocess environment for CLI models is always filtered to exclude `CLAUDECODE`, preventing nested-agent mode that would corrupt the `--output-format json` output.
- `cost-tracking-is-process-scoped-and-thread-safe` — `_cost_tracker` accumulates stats for the lifetime of the Python process and is guarded by a single `threading.Lock`; there is no per-call cost return value, only post-hoc queries via `get_cost_summary()`.
- `json-parse-failure-degrades-to-raw-output` — If a CLI model's stdout is not valid JSON, `_parse_cli_json` silently returns the raw string, meaning cost/token tracking is lost without any warning or exception.
- `openai-uses-responses-api-not-chat-completions` — The `openai:` provider calls `client.responses.create()` (the newer Responses API) and iterates `response.output` items to extract text, which is structurally different from the Chat Completions `choices[0].message.content` pattern.
