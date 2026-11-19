---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: invoke_model in reasons/llm.py

# `invoke_model` — Dispatcher for LLM Invocation

## Purpose

`invoke_model` is the single entry point for calling any supported language model in the `reasons` package. It abstracts over three very different transport mechanisms:

1. **CLI subprocess** — `claude`, `gemini`, `ollama:*` binaries piped over stdin/stdout.
2. **LangChain API adapters** — `api:*` (Anthropic) and `vertex:*` (Anthropic on Vertex).
3. **OpenAI Responses API** — `openai:*` via the `openai` SDK.

Regardless of the backend, callers get back a plain string of response text, and the module-level `_cost_tracker` accumulates token/cost/elapsed stats that callers can read via `get_cost_summary()` / `format_cost_summary()`.

## Contract

**Preconditions**
- `prompt` is a non-empty string (not enforced by type system — a subprocess or API call with an empty prompt will simply proceed).
- `model` must be one of: a name in `MODEL_COMMANDS`, a `claude:<x>`, `gemini:<x>`, `ollama:<x>`, `openai:<x>`, `api:<x>`, or `vertex:<x>` specifier. **Anything else raises `ValueError` from `resolve_model_cmd`.**
- For CLI models, the corresponding binary must exist on `PATH`.
- For `api:` / `vertex:`, the corresponding LangChain package must be importable and (for vertex) `GOOGLE_CLOUD_PROJECT` must be set.

**Postconditions**
- Returns the model's response as a string (never `None`).
- Global `_cost_tracker["elapsed_seconds"]` is incremented by the call wall-clock duration **even on failure**, because of the `finally` block.
- On success, if the backend records cost (`_parse_cli_json` for claude/gemini, `_invoke_openai` for `openai:`), token/cost stats are merged into `_cost_tracker`.
- A `by_model[model]` entry is created **only if `_record_cost` ran at least once for that exact model string**.

**Invariants**
- The timeout is passed to whichever backend is used; API timeouts are converted to `subprocess.TimeoutExpired` inside `_invoke_api` so callers have a uniform timeout exception (though `_invoke_openai` does **not** perform that conversion — it lets the raw `openai` SDK exception through).
- The env passed to the CLI subprocess never contains `CLAUDECODE`.

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `prompt` | `str` | Text piped to the CLI's stdin or sent to the API | Empty string allowed; segmented prompts are caller's responsibility |
| `model` | `str` (default `"claude"`) | Specifier selecting the backend + model. Prefix determines dispatch. | Case-sensitive; unknown → `ValueError`; `"api:"` / `"vertex:"` with empty name passes the `startswith` check but fails later in `ChatAnthropic(model="")` |
| `timeout` | `int` (default `300`) | Seconds. Passed as `timeout=` to `subprocess.run`, as `float(timeout)` to `ChatAnthropic`/`ChatAnthropicVertex`, and as `timeout=timeout` to `client.responses.create`. | `0` or negative values are not validated; behavior depends on the backend (subprocess will likely raise `TimeoutExpired` immediately) |

## Return Value

Always a `str`, in exactly one of these forms:

- **CLI + JSON-capable (claude, gemini)**: parsed response field (`data["result"]` for claude, `data["response"]` for gemini). On JSON parse failure, raw stdout is returned and **no cost is recorded**.
- **CLI ollama**: raw stdout, with an optional stripping of the `"Thinking...\n"…"...done thinking.\n"` preamble.
- **API models (`api:`, `vertex:`)**: `response.content` from the LangChain adapter.
- **OpenAI**: concatenation of `text` fields from every `message` item in `response.output`.

Callers must handle `FileNotFoundError`, `ImportError`, `RuntimeError`, `ValueError`, and `subprocess.TimeoutExpired`.

## Algorithm (step by step)

1. Capture `t0 = time.monotonic()` — *before any dispatch* so timing covers the whole call.
2. **Dispatch by prefix (order matters):**
   - `api:` or `vertex:` → delegate to `_invoke_api`, return immediately.
   - `openai:` → delegate to `_invoke_openai`, return immediately.
   - Anything else → treat as CLI.
3. Resolve the command via `resolve_model_cmd(model)`. This raises `ValueError` for unknown models.
4. `binary = cmd[0]`. Verify with `shutil.which(binary)`. **This checks the *first token only*** — if the command were multi-word (it isn't here) or used an absolute path, the check would still behave as documented.
5. Build a copy of `os.environ` *without* `CLAUDECODE`. This prevents nested Claude sessions (running inside Claude Code) from inheriting that variable into the child CLI.
6. `subprocess.run(cmd, input=prompt, capture_output=True, text=True, timeout=timeout, env=env)`.
   - `input=prompt` → prompt goes to **stdin**, not argv.
   - `text=True` → stdout/stderr are `str`, not `bytes`.
7. If `returncode != 0`, raise `RuntimeError(f"{model} failed: {result.stderr}")`.
8. If `model.startswith("ollama:")`:
   - Look for the literal string `"Thinking...\n"`.
   - If found, split on the literal `"...done thinking.\n"` with `maxsplit=1` and take the right half.
   - Return the (possibly stripped) output **directly** — no JSON parsing, no cost recording.
   - Note: the comment calls this *"Fragile"* — new Ollama versions change the markers.
9. Otherwise, pass stdout to `_parse_cli_json(output, model)`, which:
   - Attempts `json.loads`; on failure returns raw output (no cost record).
   - For `gemini*`, extracts `stats.models.*.tokens.{input,candidates}` and records cost with `total_cost_usd` forced to `0.0`.
   - Otherwise (claude), extracts `data["result"]` and sums `usage.input_tokens + cache_creation_input_tokens + cache_read_input_tokens` for input, `usage.output_tokens` for output, and `total_cost_usd` from the JSON.
10. **`finally` block** (runs on every return and every exception):
    - Computes elapsed.
    - Under `_cost_lock`, adds to `_cost_tracker["elapsed_seconds"]` (always).
    - If `model in _cost_tracker["by_model"]`, also adds to `by_model[model]["elapsed_seconds"]` — **only when that entry already exists**, which happens only if `_record_cost` ran earlier in this or a previous call.

## Side Effects

- **Spawns a subprocess** (CLI path) or **makes a network request** (`api:`, `vertex:`, `openai:`).
- **Mutates global `_cost_tracker`** under `_cost_lock`: `elapsed_seconds` always; token/cost fields only when the backend reports them.
- **Reads `os.environ`** and constructs a filtered copy for the child process.
- **Reads `PATH`** via `shutil.which`.
- May initialize the **Langfuse callback handler** lazily through `_invoke_api` → `_get_langfuse_handler` (checked once via `_langfuse_checked`).

## Error Handling

| Raised | When |
|---|---|
| `FileNotFoundError` | CLI model whose binary is not on `PATH`. |
| `ImportError` | `api:` / `vertex:` / `openai:` requested but matching package missing (raised inside delegate). |
| `RuntimeError` | CLI `returncode != 0`; or an API call raised a non-timeout exception (wrapped in `_invoke_api`). |
| `subprocess.TimeoutExpired` | CLI subprocess timed out, or (in `_invoke_api`) an API exception whose name/message contains "timeout". |
| `ValueError` | Unknown `model` (from `resolve_model_cmd`) or unknown API prefix. |

Notable gaps:
- `_invoke_openai` does **not** translate `openai.APITimeoutError` into `subprocess.TimeoutExpired` — callers must catch both.
- JSON parse failures in `_parse_cli_json` are **swallowed** — the raw stdout is returned and no cost is recorded.
- Empty `prompt` is not rejected.

## Usage Patterns

- Typically called as `invoke_model(prompt, model="claude")` or with a more specific specifier like `"claude:sonnet"`, `"gemini:gemini-2.5-flash"`, `"ollama:gemma3:4b"`, `"api:claude-sonnet-4-5"`, `"openai:gpt-4o"`.
- Callers wanting aggregated cost should call `get_cost_summary()` **after** a batch of invocations, not per call.
- Callers in long-running loops should be aware the global tracker is process-wide and never auto-resets — call `reset_cost_tracker()` between jobs if isolation is needed.
- Callers should not rely on the ollama "thinking" stripping being stable (`# Fragile` comment).

**Assumptions not enforced by types:**
- The dispatch check is strictly case-sensitive (`"Claude"` would go to `resolve_model_cmd` and raise `ValueError`).
- The env-filter only removes the exact key `"CLAUDECODE"` — any variant (e.g., `CLAUDE_CODE`, `CLAUDECODE_*`) still leaks through.
- `api:`/`vertex:` calls skip per-model elapsed tracking (see below) — an assumption that the caller's cost dashboards account for this.

## Dependencies

- **Stdlib**: `json`, `os`, `shutil`, `subprocess`, `threading`, `time` (imported locally).
- **Optional third-party**: `langchain_anthropic`, `langchain_google_vertexai.model_garden`, `langfuse.langchain`, `openai`.
- **Intra-module**: `_invoke_api`, `_invoke_openai`, `resolve_model_cmd`, `_parse_cli_json`, `_record_cost` (indirect), `_get_langfuse_handler` (indirect), `_cost_tracker`, `_cost_lock`.

---

## Topics to Explore

- [file] `reasons/llm.py` — The surrounding module provides the cost tracker, all backends, and the Ollama "thinking" stripping; reading the whole file is essential to see how `invoke_model`'s contract interacts with the counters.
- [function] `reasons/llm.py:_parse_cli_json` — This is where per-model `by_model` entries are created; the `finally` guard in `invoke_model` depends on this function having run.
- [function] `reasons/llm.py:_invoke_api` — Handles the `api:`/`vertex:` paths and the timeout-exception translation to `subprocess.TimeoutExpired`; note it never calls `_record_cost`.
- [file] `tests/test_llm.py` — Exercises dispatch, cost recording, timeouts, and ollama-specific behavior; a good cross-check for the beliefs below.
- [general] `cost-tracker-design` — The global-vs-per-model asymmetry (elapsed always, tokens/cost conditionally, per-model elapsed only after a `_record_cost`) is a subtle design choice worth designing against for any new backend.

## Beliefs

- `invoke_model-dispatches-by-model-prefix` — The `model` string's prefix (`api:`, `vertex:`, `openai:`, otherwise CLI) selects the transport backend.
- `invoke_model-records-elapsed-in-finally` — `_cost_tracker["elapsed_seconds"]` is incremented even when the call raises, because the increment is in a `finally` block.
- `invoke_model-strips-claudecode-env` — The environment passed to CLI subprocesses omits the `CLAUDECODE` key.
- `ollama-models-skip-cost-tracking` — `ollama:*` responses return before `_parse_cli_json`, so they never call `_record_cost` and never appear in `by_model`.
- `api-models-skip-per-model-elapsed` — `api:` and `vertex:` calls never create a `by_model` entry, so their per-model `elapsed_seconds` is never accumulated (only the global counter).
- `per-model-elapsed-requires-prior-cost-record` — The `if model in _cost_tracker["by_model"]` guard means per-model elapsed time only accumulates after a successful `_record_cost` created the entry.
