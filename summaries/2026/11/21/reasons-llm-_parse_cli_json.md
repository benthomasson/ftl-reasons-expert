---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _parse_cli_json in reasons/llm.py

# `_parse_cli_json` — Explanation

## 1. Purpose

Normalizes the raw stdout of a CLI LLM tool (`claude` or `gemini`) into a plain text response while simultaneously recording token/cost statistics into the module's global cost tracker.

The function exists because `invoke_model` invokes each CLI with `--output-format json`. That JSON envelope contains both the model's response text **and** usage metadata (token counts, and for Claude a `total_cost_usd`). Rather than let every caller re-parse the envelope, this helper centralizes:

1. **Response extraction** — pull out the LLM's actual text.
2. **Cost accounting** — feed `_record_cost(...)` so `get_cost_summary()` / `format_cost_summary()` work.

It is deliberately defensive: anything unexpected (non-JSON output, non-dict JSON, missing keys) degrades to returning the raw string so the caller still receives *something* usable.

## 2. Contract

- **Precondition**: `output` is the complete stdout of a CLI process invoked in the shape produced by `resolve_model_cmd` / `MODEL_COMMANDS`. `model` is a string identifying which CLI it was.
- **Postcondition (success path)**: the response text is returned, and `_record_cost(model, input_tokens, output_tokens, cost_usd)` has been called exactly once, which increments the global call counter.
- **Postcondition (failure path)**: `output` is returned unchanged, and **no** cost is recorded for that call (but `elapsed_seconds` is still updated by `invoke_model`'s `finally`).
- **Invariant**: The function never raises on malformed JSON, and never raises when the parsed JSON is not a `dict`. It *can* still raise `AttributeError`/`TypeError` if a parsed dict contains fields of unexpected types (see Error Handling).
- **Assumption not enforced by types**: `data["response"]`, `data["result"]`, `data["stats"]`, `data["usage"]`, and `data["total_cost_usd"]` are of the expected types when present.

## 3. Parameters

| Name | Type | Description |
|------|------|-------------|
| `output` | `str` | Raw stdout from the CLI subprocess. May be empty, may be non-JSON text (e.g., if the model ignored `--output-format json` or the CLI failed silently), may be JSON that isn't the expected envelope. |
| `model` | `str` | The model identifier used to select the envelope schema. `"gemini"` / `"gemini:<submodel>"` → Gemini schema; anything else → Claude schema. This prefix match is broader than `resolve_model_cmd`'s (which requires exact `"gemini"` or `"gemini:<submodel>"`), so a hypothetical `"gemini-foo"` would take the Gemini branch even though `resolve_model_cmd` would reject it. |

**Edge case**: The condition `model.startswith("gemini") or model.startswith("gemini:")` — the second clause is **unreachable**. Any string starting with `"gemini:"` already starts with `"gemini"`, so `"gemini".startswith("gemini")` short-circuits. The redundant clause is harmless but dead code. (`except (json.JSONDecodeError, ValueError)` is similarly redundant, since `JSONDecodeError` subclasses `ValueError`.)

## 4. Return Value

Returns `str` in three cases:

- **Parse failure / non-dict JSON**: `output` verbatim.
- **Gemini branch**: `data["response"]` if present, otherwise `output`. (The `.get(..., output)` fallback means a genuinely missing `response` key yields the raw JSON — arguably less useful, but consistent with the "fall back" philosophy.)
- **Claude branch**: `data["result"]` if present, otherwise `output`.

**Caveat**: The declared return type is `str`, but nothing validates that `data["response"]` or `data["result"]` is a string. If a CLI ever emits `{"result": 42}`, the function returns `42` (an `int`), which will break callers that assume `.strip()`, concatenation, etc. There is also no guard against `{"result": null}` — that would return `None`.

The caller (`invoke_model`) simply returns whatever this function yields.

## 5. Algorithm

1. **Attempt parse** — `json.loads(output)`.
   - On `JSONDecodeError`/`ValueError`: return `output` unchanged. **No cost recorded.**
2. **Shape guard** — if the parsed value is not a `dict` (e.g., a bare string, list, number, or `null`), return `output`. **No cost recorded.**
3. **Branch on model family:**

   **Gemini** (`model.startswith("gemini")`):
   - `text = data.get("response", output)`.
   - `stats = data.get("stats", {})` — expected shape: `{"models": {<name>: {"tokens": {"input": N, "candidates": M}}, ...}}`.
   - Initialize `input_tokens = output_tokens = 0`.
   - Iterate every model entry under `stats["models"]`, summing `tokens.input` into `input_tokens` and `tokens.candidates` into `output_tokens`. Missing keys default to `0` via `.get(..., 0)`.
   - Call `_record_cost(model, input_tokens, output_tokens, 0.0)` — Gemini CLI doesn't emit a dollar amount, so `cost_usd` is hardcoded `0.0`.
   - Return `text`.

   **Claude / default**:
   - `text = data.get("result", output)`.
   - `usage = data.get("usage", {})`.
   - `input_tokens = usage.input_tokens + usage.cache_creation_input_tokens + usage.cache_read_input_tokens` — all three are collapsed into a single "input" number (each defaults to `0`).
   - `output_tokens = usage.output_tokens` (default `0`).
   - `cost_usd = data.total_cost_usd` (default `0.0`).
   - Call `_record_cost(model, input_tokens, output_tokens, cost_usd)`.
   - Return `text`.

**Subtle point**: `_record_cost` is called *even when* the parsed dict lacks the expected `response`/`result` keys (because the code reaches the record call regardless of whether it fell back to `output`). The cost call still bumps `_cost_tracker["calls"]` — a valid JSON dict of an unrelated shape will be counted as a 0-token, 0-cost LLM call. This is intentional defensiveness in spirit, but has an observable side effect on `format_cost_summary()`.

## 6. Side Effects

- Calls `_record_cost(...)`, which:
  - Acquires `_cost_lock` (a module-level `threading.Lock`).
  - Mutates the global `_cost_tracker` dict: increments `calls`; adds to `input_tokens`, `output_tokens`, `total_cost_usd`; and updates or creates a per-model entry under `by_model`. That per-model entry also gets `calls`, `input_tokens`, `output_tokens`, and `total_cost_usd` incremented; `elapsed_seconds` is left for `invoke_model` to update.
- No file, network, or subprocess I/O of its own.
- No stdout/stderr writing.

## 7. Error Handling

- **Swallowed**: `json.JSONDecodeError` and `ValueError` (the latter covers the former). Any other exception during `json.loads` would propagate, but in practice only `TypeError` (if `output` weren't a `str`) is conceivable.
- **Not caught**:
  - If `data["stats"]` is a non-dict (e.g., `[]`, `"x"`), `stats.get("models", {})` raises `AttributeError`.
  - If `stats["models"]` exists but a value is not a dict, `.get("tokens", {})` on that value raises `AttributeError`.
  - If `data["usage"]` is a non-dict, `usage.get("input_tokens", 0)` raises `AttributeError`.
  - If `usage.get(...)` returns a non-`int` (e.g., a string), `+` raises `TypeError`.
  - If `data["total_cost_usd"]` is a non-`float` (e.g., a string), `_record_cost` will raise `TypeError` when doing `+=` into the tracker.

  These downsides are the cost of "assume the CLI schema is right once we've confirmed it's a dict" — the code trusts the *envelope structure* unconditionally once `isinstance(data, dict)` passes.

## 8. Usage Patterns

The only caller is `invoke_model`:

```python
output = result.stdout
# ... ollama thinking-marker stripping ...
if model.startswith("ollama:"):
    return output
return _parse_cli_json(output, model)
```

**Caller obligations:**

- Must *not* be called for `ollama:` models — Ollama doesn't produce JSON, and `_parse_cli_json` would silently double-return raw output (with no cost recorded, since parse fails). `invoke_model` bypasses it.
- Must *not* be called for `api:`, `vertex:`, or `openai:` models — those paths never reach it (handled earlier in `invoke_model` via `_invoke_api` / `_invoke_openai`, which have their own `_record_cost` calls).
- The caller does not need to handle exceptions from this function under normal-operation assumptions, but must tolerate `None` or non-`str` return values if a future CLI version emits a different JSON shape.

**Reliance on side effects**: `invoke_model` depends on this function to record cost for CLI models. If someone added a new CLI model family that returned JSON without reaching a `_record_cost` call, the token/money counter would silently under-report.

## 9. Dependencies

- **Stdlib**: `json` (`json.loads`, `json.JSONDecodeError`).
- **Internal**: `_record_cost` (defined above in the same module) — the sole coupling point to the module state.
- **Implicit**: `MODEL_COMMANDS`-driven CLI invocations dictating the envelope structure:
  - `claude -p --output-format json` → `{"result": str, "usage": {...}, "total_cost_usd": float, ...}`
  - `gemini --skip-trust -o json -p ""` → `{"response": str, "stats": {"models": {...}}, ...}`

---

## Topics to Explore

- [function] `reasons/llm.py:_record_cost` — Where the parsed tokens actually land; understand the `by_model` accumulation and threading semantics.
- [function] `reasons/llm.py:invoke_model` — Full pipeline showing when `_parse_cli_json` is bypassed (Ollama) vs. used (Claude/Gemini), and how elapsed time is always recorded via `finally`.
- [file] `tests/test_llm.py` — Existing coverage for the Gemini vs. Claude branches, fallback behavior, and malformed JSON.
- [general] `gemini-cli-json-schema` — The `stats.models.<name>.tokens.input` / `.candidates` shape is defined by Gemini's CLI, not this repo; verify stability across versions.
- [general] `claude-cli-json-schema` — Similarly, `result` / `usage` / `cache_creation_input_tokens` / `cache_read_input_tokens` / `total_cost_usd` are dictated by `claude -p --output-format json`.

## Beliefs

- `parse-falls-back-to-raw-output-on-parse-failure` — If `json.loads` raises or the parsed value is not a dict, `output` is returned verbatim.
- `parse-records-cost-even-when-response-missing` — Once the parsed value is a dict, `_record_cost` is always invoked, incrementing `calls` even if the expected `response`/`result` key is absent.
- `gemini-prefix-check-is-redundant` — The `or model.startswith("gemini:")` clause is unreachable because `"gemini:..."` already starts with `"gemini"`.
- `claude-branch-collapses-cache-tokens-into-input-tokens` — `cache_creation_input_tokens` and `cache_read_input_tokens` are added to `input_tokens`, so reported token breakdown does not distinguish cache hits from fresh input.
- `parse-assumes-string-response-payload` — The return type is annotated `str` but not enforced; a JSON dict with a non-string `result`/`response` would return that value unchanged.
