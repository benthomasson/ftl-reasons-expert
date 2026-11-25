---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Similarly, `result` / `usage` / `cache_creation_input_tokens` / `cache_read_input_tokens` / `total_cost_usd` are dictated by `claude -p --output-format json`.

## How `claude -p --output-format json` dictates the result/usage/cost fields

The code path is short and mechanical: the CLI is *told* to emit JSON, and a single function (`_parse_cli_json`) does all field extraction.

### 1. The command is what requests the JSON envelope

In `reasons/llm.py`:

- `MODEL_COMMANDS` (line 50) maps `"claude"` to `["claude", "-p", "--output-format", "json"]`.
- `resolve_model_cmd` (line ~185) does the same for submodels: `["claude", "-p", "--model", submodel, "--output-format", "json"]`.

Tests pin this contract:
- `tests/test_llm.py:27` — `resolve_model_cmd("claude") == ["claude", "-p", "--output-format", "json"]`
- `tests/test_llm.py:49` — same for `claude:sonnet`

So the *shape* of the stdout envelope (the keys `result`, `usage`, `total_cost_usd`) is imposed entirely by the Claude CLI's `--output-format json` mode; nothing in this repo constructs it.

### 2. `_parse_cli_json` folds the envelope into internal fields

`reasons/llm.py` (`_parse_cli_json`, roughly lines 130–155) does:

```python
data = json.loads(output)          # CLI stdout

text          = data.get("result", output)           # ← returned response body
usage         = data.get("usage", {})                # ← dict from CLI
input_tokens  = usage.get("input_tokens", 0) \
              + usage.get("cache_creation_input_tokens", 0) \
              + usage.get("cache_read_input_tokens", 0)
output_tokens = usage.get("output_tokens", 0)
cost_usd      = data.get("total_cost_usd", 0.0)      # ← top-level, NOT inside usage

_record_cost(model, input_tokens, output_tokens, cost_usd)
return text
```

Field mapping is therefore:

| Claude JSON field | Internal handling |
|---|---|
| `result` | returned as the model response string (falls back to raw `output`) |
| `usage` | source dict for all token counts |
| `usage.input_tokens` | base input tokens |
| `usage.cache_creation_input_tokens` | **added into** `input_tokens` (cache *writes* count as input) |
| `usage.cache_read_input_tokens` | **added into** `input_tokens` (cache *hits* count as input) |
| `usage.output_tokens` | output token count |
| `total_cost_usd` (top level) | accumulated into the cost tracker |

Two important consequences of this flattening:

- **Cache accounting is lost in the aggregate.** `cache_creation_input_tokens` and `cache_read_input_tokens` are summed with `input_tokens`, so `_cost_tracker` cannot report cache hit ratio. Only the raw parsed `usage` dict (which is discarded) had that detail.
- **`total_cost_usd` is read from the envelope root, not from `usage`.** This is enforced by the mocks in the tests: `json.dumps({"result": "response", "usage": {}, "total_cost_usd": 0.0})` at `tests/test_llm.py:67`, `:121`, `:129`, `:172`.

### 3. `_record_cost` is the only consumer, into `_cost_tracker`

`reasons/llm.py` (`_record_cost`, ~lines 105–125) mutates the module-level `_cost_tracker` dict (initialised at line ~60 with `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `elapsed_seconds`, `by_model`). `get_cost_summary()` and `format_cost_summary()` (lines ~80–100) expose it; the CLI somewhere serialises it into `costs/*.json`.

Evidence that the aggregate fields are derived the same way:
- `tests/test_llm.py:342-351` — a CLI payload with `usage.input_tokens=100`, `output_tokens=50`, `total_cost_usd=0.0021` produces `s["total_cost_usd"] == pytest.approx(0.0021)`.
- `tests/test_llm.py:357-363` — a payload with `cache_creation_input_tokens=200` and `cache_read_input_tokens=100` plus `input_tokens=100` should yield an aggregated input token count of 400 (matching the sum in `_parse_cli_json`).
- `costs/2026-11-19T180331-update.json` shows the persisted shape (`input_tokens`, `output_tokens`, `cost_dollars`, `by_model[*].total_cost_usd`) — that record happens to be an `openai:` call, so its cost is `0.0`, but the schema is identical.

### 4. Fallbacks: when the CLI doesn’t emit the expected shape

- If `json.loads(output)` raises `JSONDecodeError`/`ValueError`, `_parse_cli_json` returns the raw `output` **without recording any cost** (lines ~135–139). A claude call that somehow emits prose is silently treated as zero-cost text.
- If `data` is not a `dict`, same fallback.
- If `data` is a dict but missing keys, every `.get(..., 0)` defaults to zero — no exception, but the cost tracker under-counts.
- Non-CLI back-ends diverge here: gemini goes through the same function but reads `data["response"]` and `stats.models[*].tokens` (input/candidates) instead; `api:`/`vertex:` models never hit `_parse_cli_json` at all — they use `_invoke_api` and read `response.usage.input_tokens`/`output_tokens` (lines ~269–270), with **no** cache token fields and **no** `total_cost_usd` at all.

### 5. What the observation set does *not* show

The JSON schema emitted by the actual `claude` binary isn’t in the repo — only the *consumers* of it. The exact set of keys, and whether cache fields exist on every plan/tier, is defined upstream by the Claude CLI, not by this codebase.

## Topics to Explore

- [function] `reasons/llm.py:_parse_cli_json` — The single point where `result`/`usage`/`total_cost_usd` are extracted; everything else downstream is just aggregation.
- [function] `reasons/llm.py:resolve_model_cmd` — Where the `--output-format json` argv is constructed for both `claude` and `claude:<submodel>`.
- [file] `tests/test_llm.py` — Round-trips the assumed CLI JSON shape; lines 342–363 encode the cache-token summation rule.
- [file] `costs/2026-11-19T180331-update.json` — Persisted form of `_cost_tracker`, showing how token counts and `total_cost_usd` are surfaced to users.
- [general] `claude-cli-json-schema` — The upstream, external contract this repo depends on; not documented in-tree.

## Beliefs

- `claude-cli-uses-output-format-json` — Both `claude` and `claude:<submodel>` argv end in `--output-format json`, so the envelope keys are dictated by the CLI, not by this repo.
- `parse-cli-json-extracts-result-and-cost` — `_parse_cli_json` returns `data["result"]` and records `data["total_cost_usd"]` plus summed token counts via `_record_cost`.
- `cache-tokens-fold-into-input-tokens` — `cache_creation_input_tokens` and `cache_read_input_tokens` are added to `usage.input_tokens`, so the aggregate tracker cannot distinguish cache hits/writes.
- `total-cost-usd-is-top-level` — `total_cost_usd` is read from the JSON root, not from `usage`, matching the mocks in `tests/test_llm.py:67,121,129,172`.
- `non-json-claude-output-skips-cost-recording` — If `json.loads` fails, `_parse_cli_json` returns raw text and no `_record_cost` call happens for that invocation.
