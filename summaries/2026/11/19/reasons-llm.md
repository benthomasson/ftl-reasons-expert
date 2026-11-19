---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/llm.py

# `reasons/llm.py` — Shared LLM Invocation Layer

## Purpose

This module is the **single abstraction point for calling language models** across the entire `reasons` project. Every other module that needs an LLM (`ask.py`, `verify.py`, `repair.py`, `contradictions.py`, etc.) calls `invoke_model()` rather than shelling out or hitting an SDK directly. The file owns four responsibilities:

1. **Model routing** — parsing a model *specifier* string (`claude`, `gemini`, `claude:sonnet`, `ollama:gemma3:4b`, `api:claude-sonnet-4`, `vertex:...`, `openai:...`) into a concrete transport (CLI subprocess vs. API adapter).
2. **Transport execution** — running the CLI subprocess or the LangChain/OpenAI SDK call.
3. **Cost/token accounting** — accumulating per-call stats across the process lifetime so a caller can report session cost.
4. **Optional observability** — wiring Langfuse tracing into LangChain calls when configured.

It deliberately keeps the public surface tiny: `invoke_model()` plus a handful of cost-reporting helpers.

---

## Key Components

### Constants

| Name | Role |
|---|---|
| `MODEL_COMMANDS` | Dispatch table mapping short model aliases (`"claude"`, `"gemini"`) to base CLI command lists. Both use JSON output format so costs can be parsed. |
| `_cost_tracker` | Global mutable dict aggregating `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `elapsed_seconds`, and a `by_model` sub-dict. |
| `_cost_lock` | `threading.Lock` guarding `_cost_tracker`, so the module is safe to use from multiple threads. |
| `_langfuse_handler`, `_langfuse_checked` | Process-wide lazy singleton for the Langfuse callback handler — created at most once. |

### Optional-dependency flags

The module uses the classic try/except-import idiom:

- `_HAS_LANGCHAIN_ANTHROPIC`, `_HAS_LANGCHAIN_VERTEX`, `_HAS_LANGFUSE`, `_HAS_OPENAI`

Each import block sets the symbol to `None` on `ImportError`, allowing the module to load cleanly without the optional extras. This means `reasons/llm.py` can be imported on a bare-bones install (CLI-only); API features fail *at call time*, not import time.

### Public functions

- **`invoke_model(prompt, model="claude", timeout=300) -> str`** — the main entry point. Dispatches to `_invoke_api`, `_invoke_openai`, or the CLI subprocess path based on the model prefix. Records elapsed time in a `finally` block so even failures contribute to timing stats.
- **`resolve_model_cmd(model) -> list[str]`** — pure function turning a CLI model spec into an argv list. Raises `ValueError` with an availability list on unknown specs.
- **`reset_cost_tracker()`** — clears the global counters (used in tests, typically in a fixture).
- **`get_cost_summary() -> dict`** — returns a **deepcopy** of the tracker so callers can't mutate internal state.
- **`format_cost_summary() -> str`** — human-readable string; returns `""` if no calls have been made (so callers can concat it safely).

### Private helpers

- **`_invoke_api(prompt, model, timeout)`** — splits `api:` / `vertex:`, builds the appropriate LangChain chat model, attaches the Langfuse handler if available, and normalizes exception types (timeouts → `subprocess.TimeoutExpired`, anything else → `RuntimeError`).
- **`_invoke_openai(prompt, model, timeout)`** — uses the OpenAI **Responses API** (`client.responses.create`), walks `response.output[*].content[*].text` to concatenate text, and records tokens. Note it does *not* record a dollar cost (passes `0.0`).
- **`_parse_cli_json(output, model)`** — parses the CLI's JSON stdout envelope. For Gemini it reads `data["response"]` and sums tokens from `stats["models"][*].tokens.{input,candidates}`. For Claude it reads `data["result"]` and sums `usage.input_tokens + cache_creation_input_tokens + cache_read_input_tokens`, and uses `data["total_cost_usd"]`. Falls back to raw output on any non-dict / parse failure — a deliberate "fail-open" so callers get *something* rather than an exception.
- **`_record_cost(model, input_tokens, output_tokens, cost_usd)`** — atomic increments to both aggregate and per-model counters under `_cost_lock`.
- **`_get_langfuse_handler()`** — lazy singleton; returns `None` if the package is absent or `LANGFUSE_PUBLIC_KEY` isn't set.

---

## Patterns

1. **Strategy dispatch by string prefix.** The `model:submodel` convention (`api:`, `vertex:`, `openai:`, `ollama:`, `claude:`, `gemini:`) acts as a scheme identifier. Adding a new provider means adding a branch in `invoke_model`/`resolve_model_cmd` and a `_invoke_*` helper — no caller changes.
2. **Optional-dependency guard.** Every heavyweight import is wrapped in try/except and gated behind `_HAS_*` booleans. This is the classic "extras" pattern (`pip install 'ftl-reasons[api]'`).
3. **Global mutable state behind a lock.** `_cost_tracker` is a process-wide singleton (a module-level dict). The lock is short-lived and only wraps counter mutation — the LLM call itself is *not* held under lock, which is essential for parallelism.
4. **Fail-open parsing.** `_parse_cli_json` returns raw output when JSON parsing fails, preserving the caller's ability to get text even if a CLI changes its JSON envelope.
5. **Exception normalization at the boundary.** The API path translates provider-specific timeout exceptions into `subprocess.TimeoutExpired` and everything else into `RuntimeError` — so callers see a uniform error vocabulary regardless of transport.
6. **Time in `finally`.** Elapsed time is recorded even when `invoke_model` raises, so `format_cost_summary` reports honest wall-clock time including failures.

---

## Dependencies

**Inbound (imports this file makes):**
- Stdlib: `json`, `os`, `shutil`, `subprocess`, `threading`, `copy`, `time`.
- Optional third-party: `langchain_anthropic.ChatAnthropic`, `langchain_google_vertexai.model_garden.ChatAnthropicVertex`, `langfuse.langchain.CallbackHandler`, `openai`.

**Outbound (modules that import this):**
- Core commands: `reasons/api.py`, `reasons/ask.py`, `reasons/verify.py`, `reasons/repair.py`, `reasons/repair_premises.py`, `reasons/review.py`, `reasons/review_justifications.py`, `reasons/review_premises.py`, `reasons/propose_update.py`, `reasons/report_belief.py`, `reasons/report_gated.py`, `reasons/contradictions.py`, `reasons/build_wiki.py`, `reasons/cli.py`, `reasons/pg.py`.
- Note also `reasons_lib/llm.py` — this suggests a *mirrored* package layout (`reasons/` and `reasons_lib/`). `reasons_lib/llm.py` likely imports this or is a synchronization shim.

The broad inbound fan-in confirms this is a **low-level utility module** — it sits at the bottom of the dependency graph and is deliberately dependency-light.

---

## Flow

### CLI path (`invoke_model` with e.g. `"claude"` or `"ollama:..."`)

```
invoke_model(prompt, "claude:sonnet")
  └─ resolve_model_cmd → ["claude", "-p", "--model", "sonnet", "--output-format", "json"]
     └─ shutil.which("claude") → FileNotFoundError if missing
     └─ subprocess.run(cmd, input=prompt, capture_output=True, ...)
        env strips CLAUDECODE (to avoid recursive-nesting guard in Claude CLI)
        └─ non-zero returncode → RuntimeError(stderr)
     └─ if ollama: strip "Thinking...\n...done thinking.\n" preamble
     └─ if not ollama: _parse_cli_json → text + _record_cost
```

The `env` filter that removes `CLAUDECODE` is important: it **prevents the Claude CLI from refusing to run inside an existing Claude Code session**, which is a common failure mode when this tool is invoked by an agent.

### API path (`api:` or `vertex:`)

```
invoke_model(prompt, "api:claude-sonnet-4")
  └─ _invoke_api
     ├─ prefix == "api" → ChatAnthropic(model=name, timeout=...)
     ├─ prefix == "vertex" → ChatAnthropicVertex(project from GOOGLE_CLOUD_PROJECT, ...)
     ├─ build config with optional Langfuse callback
     ├─ llm.invoke(prompt, config=config)
     └─ timeout-like exceptions → subprocess.TimeoutExpired; others → RuntimeError
```

Note: the API path **does not** call `_record_cost` — only the OpenAI Responses path and CLI JSON path do. That is a real gap a maintainer should know about.

### OpenAI path (`openai:`)

```
invoke_model(prompt, "openai:gpt-4o")
  └─ _invoke_openai
     ├─ OpenAI() client (relies on OPENAI_API_KEY env)
     ├─ responses.create(model=..., input=prompt, timeout=...)
     ├─ walk output[*].content[*] collecting .text
     └─ _record_cost(model, input_tokens, output_tokens, 0.0)
```

---

## Invariants

1. **`_cost_tracker` mutations always happen under `_cost_lock`.** Every write site (`reset_cost_tracker`, `_record_cost`, `invoke_model`'s `finally`) uses the lock.
2. **`get_cost_summary()` returns a deepcopy**, so external mutation of the returned dict cannot corrupt the tracker. `format_cost_summary()` reads under the lock but formats within it — no deadlock risk since the lock is non-reentrant and not nested.
3. **API-prefixed models never reach `resolve_model_cmd`.** `invoke_model` dispatches on `api:` / `vertex:` / `openai:` before calling `resolve_model_cmd`, and `resolve_model_cmd` explicitly rejects `openai:` with a `ValueError` pointing users to the API path.
4. **A model must match exactly one of: a key in `MODEL_COMMANDS`, or one of the known prefixes.** Otherwise `resolve_model_cmd` raises `ValueError` including the list of valid forms.
5. **`CLAUDECODE` is unconditionally removed** from the subprocess env — callers cannot inject it back through this API.
6. **`invoke_model` always returns a `str`** on success, never a structured object. Parsing into richer structures is the caller's job.
7. **`format_cost_summary()` returns `""` for zero calls** — no leading `"Cost:"` prefix is emitted unless at least one call occurred (guards callers who string-concat it).

---

## Error Handling

The module is **typed to raise**, not to swallow — except for one intentional fail-open:

| Condition | Behavior |
|---|---|
| Optional dep missing for `api:`/`vertex:`/`openai:` | `ImportError` with install hint (`pip install 'ftl-reasons[...]'`). |
| `vertex:` without `GOOGLE_CLOUD_PROJECT` | `RuntimeError` with explicit env var name. |
| CLI binary absent (`shutil.which` returns `None`) | `FileNotFoundError("'{binary}' CLI not found in PATH")`. |
| CLI exits non-zero | `RuntimeError(f"{model} failed: {result.stderr}")` — stderr is surfaced verbatim. |
| CLI times out | `subprocess.TimeoutExpired` propagates from `subprocess.run`. |
| API call times out | Caught and re-raised as `subprocess.TimeoutExpired` (so callers only need to handle one timeout type). |
| API call fails otherwise | Wrapped in `RuntimeError(f"{model} failed: {exc}")` preserving `__cause__`. |
| CLI output is malformed JSON | **Swallowed** — `_parse_cli_json` returns raw output. Intended graceful degradation. |
| Langfuse not configured | Silently returns `None` — no tracing, no error. |
| Model spec unknown | `ValueError` listing valid forms. |

Note that even when `invoke_model` raises, the `finally` block still records elapsed time — the cost tracker accumulates `elapsed_seconds` for wall-clock accounting even for failed calls (though `input_tokens`/`output_tokens`/`calls` are only incremented on success by `_record_cost`).

---

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — The single facade entry point; check how every caller in the repo uses it (and whether any pass non-default timeouts).
- [file] `reasons_lib/llm.py` — Understand why the repository mirrors `reasons/` as `reasons_lib/`; the llm module is duplicated, so drift between the two is a risk.
- [file] `tests/test_llm.py` — Reveals the mocked subprocess/SDK contract and which behaviors (cost parsing, env filtering) are asserted.
- [file] `reasons/cli.py` — Where `format_cost_summary()` is likely surfaced to users; see how cost tracking is exposed at the command layer.
- [general] `optional-extras-packaging` — Inspect `pyproject.toml` for the `[api]`, `[vertex]`, `[openai]` extras that gate every `_HAS_*` flag in this file.

## Beliefs

- `llm-invoke-model-is-single-facade` — All LLM calls in the `reasons` package route through `invoke_model()`; no other module imports the SDKs directly.
- `llm-requires-lock-for-cost-tracking` — Every write to `_cost_tracker` occurs under `_cost_lock`, thread-safety is enforced at the module level.
- `openai-prefix-bypasses-resolve-model-cmd` — `resolve_model_cmd` deliberately raises for `openai:` models, which are dispatched earlier in `invoke_model` via `_invoke_openai` and the Responses API.
- `api-prefix-does-not-record-cost` — `_invoke_api` returns content without calling `_record_cost`, so `api:` and `vertex:` calls are invisible to `get_cost_summary()`.
- `cli-json-parse-fails-open` — `_parse_cli_json` returns raw output on any JSON parse failure, ensuring callers always receive a string response.
