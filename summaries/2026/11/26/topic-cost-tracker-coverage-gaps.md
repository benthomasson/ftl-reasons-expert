---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Reason about why `api:`, `vertex:`, and `ollama:` models are currently absent from `by_model` and whether that's intentional.

# Why `api:`, `vertex:`, and `ollama:` never appear in `by_model`

## The short answer

There is exactly **one** place in the codebase that creates an entry in `_cost_tracker["by_model"]`: `_record_cost()` at `reasons/llm.py:111-117`.

```python
if model not in _cost_tracker["by_model"]:
    _cost_tracker["by_model"][model] = {...}
```

Every other line that touches `by_model` is either initialization (`reasons/llm.py:65`), reset (`reasons/llm.py:77`), read-only iteration (`reasons/llm.py:325-326`), or a test assertion. So the real question is: **which code paths call `_record_cost`?**

| Path | Calls `_record_cost`? | Result |
|---|---|---|
| `claude` / `claude:<model>` (CLI) | Yes — via `_parse_cli_json` | Entered into `by_model` |
| `gemini` / `gemini:<model>` (CLI) | Yes — via `_parse_cli_json` | Entered into `by_model` |
| `openai:<model>` | Yes — directly in `_invoke_openai` | Entered into `by_model` (cost always `0.0`) |
| `ollama:<model>` | **No** | Absent |
| `api:<model>` | **No** | Absent |
| `vertex:<model>` | **No** | Absent |

The absence is a downstream consequence of two separate code decisions, and only **one** of them looks deliberate.

---

## Why `ollama:` is absent — mostly intentional

Two explicit choices route around the tracking machinery:

1. **No JSON output format.** `resolve_model_cmd()` (`reasons/llm.py:189-191`) builds `["ollama", "run", ollama_model]` — it does *not* pass `--output-format json` the way the `claude` and `gemini` entries in `MODEL_COMMANDS` do. There is no structured payload to parse, so there are no token counts to feed `_record_cost`.

2. **Explicit early return.** In `invoke_model()`, the ollama branch returns raw stdout *before* `_parse_cli_json` is ever reached (`reasons/llm.py:318-319`):

```python
if model.startswith("ollama:"):
    return output
```

   This is a deliberate guard: piping plain-text ollama output into `_parse_cli_json` would just fail `json.loads` and fall back to raw output anyway (`reasons/llm.py:120-123`), but the explicit return also skips the "Thinking...\n" handling path and makes intent clear.

So ollama is treated as a **free, local, unstructured** provider. Skipping `by_model` is consistent with that — but note the side effect: because `_record_cost` is never called, ollama calls also don't increment the *global* `calls`, `input_tokens`, or `output_tokens` counters. Only `elapsed_seconds` is captured (see below). If someone later wanted local token stats, this design would need to change.

---

## Why `api:` and `vertex:` are absent — looks like a gap, not a decision

`_invoke_api()` (`reasons/llm.py:203-260`) is dispatched from `invoke_model()` at `reasons/llm.py:289-290` and returns `response.content` directly:

```python
response = llm.invoke(prompt, config=config)
...
return response.content
```

There is **no call to `_record_cost`** anywhere in that function — no reading of `response.usage_metadata` / `response.response_metadata`, no cost accumulation, nothing. The LangChain `ChatAnthropic` / `ChatAnthropicVertex` responses do carry usage metadata, so the data is available; the code simply doesn't use it.

Compare with `_invoke_openai()` (`reasons/llm.py:263-286`), which was clearly written with tracking in mind:

```python
input_tokens = response.usage.input_tokens if response.usage else 0
output_tokens = response.usage.output_tokens if response.usage else 0
_record_cost(model, input_tokens, output_tokens, 0.0)
```

And the module docstring scopes the feature narrowly (`reasons/llm.py:9-10`):

> *Cost tracking: CLI models use `--output-format json` to capture token counts and costs.*

Nothing in the file says "API providers are excluded from `by_model`." The `_invoke_openai` precedent shows the intended pattern for non-CLI providers is to record — with cost `0.0` when a price isn't known. That makes `api:`/`vertex:` look like an **unfinished branch** rather than an intentional exclusion.

The persisted snapshot `costs/2026-11-19T180331-update.json` reinforces this: `openai:rits/deepseek-ai/...` *does* appear under `by_model` with `calls`, `input_tokens`, `output_tokens`, and `total_cost_usd: 0.0`. So non-CLI providers can and do land in `by_model` — just only via the openai path.

---

## The `elapsed_seconds` interaction (an asymmetry worth knowing)

The `finally` block in `invoke_model()` (`reasons/llm.py:321-326`) is the only other writer to `by_model`:

```python
with _cost_lock:
    _cost_tracker["elapsed_seconds"] += elapsed
    if model in _cost_tracker["by_model"]:
        _cost_tracker["by_model"][model]["elapsed_seconds"] += elapsed
```

Note the guard: `if model in _cost_tracker["by_model"]`. This is a **read-then-write on an entry that must already exist**. For `api:`, `vertex:`, and `ollama:`, the entry was never created, so per-model elapsed time is silently dropped even though the *global* `elapsed_seconds` counter does increment. The result is a tracker where `sum(by_model[*].elapsed_seconds) < elapsed_seconds` for any workload using these providers — an observable inconsistency.

---

## Test coverage confirms the orientation

`tests/test_llm.py` exercises `by_model` only for CLI providers:

- `tests/test_llm.py:434` — asserts `s["by_model"] == {}` for an empty/no-call scenario.
- `tests/test_llm.py:445-452` — `test_by_model_tracking` asserts `"claude"` and `"gemini"` keys and their token counts.

There is **no test asserting that `api:`/`vertex:`/`ollama:` are excluded** from `by_model`, and no test asserting they are included. In other words, the current behaviour is unconstrained by the test suite — if someone added `_record_cost` calls to `_invoke_api`, nothing in the tests shown here would break.

---

## Verdict

- **`ollama:` — intentional (as a side effect of design).** Local models are free and the CLI has no JSON mode, so `_record_cost` is bypassed by design. The gap is that token counts are lost entirely, not just costs.
- **`api:` / `vertex:` — almost certainly unintentional.** The dispatch and LangChain integration were added, but the tracking hook was never wired in, unlike the sibling `_invoke_openai` path. The docstring's "CLI models" phrasing is stale relative to the openai implementation.
- **Consistency risk:** because the `finally` block only updates existing keys, the tracker's per-model elapsed totals already don't reconcile with the global total whenever these providers are used.

### What's missing from the observations
- The code that serializes `_cost_tracker` into `costs/*.json` (the "update" operation) — the snapshot's `by_model` entries lack the `elapsed_seconds` field that `_record_cost` creates (`reasons/llm.py:113-116`), so the writer may project/filter keys. Without it, we can't say whether a persisted entry would even round-trip.
- The full `tests/test_llm.py` (only fragments are shown) — to confirm no hidden assertions about provider coverage exist.
- Git history / recent diffs on `reasons/llm.py` — the cheapest way to distinguish "deliberately deferred" from "forgotten".

## Topics to Explore

- [function] `reasons/llm.py:_invoke_api` — The exact spot where `_record_cost` would need to be added, along with reading `response.usage_metadata` from the LangChain `AIMessage`
- [function] `reasons/llm.py:invoke_model` — The `finally` block's `if model in _cost_tracker["by_model"]` guard is the source of the elapsed-time asymmetry; consider whether it should upsert instead
- [file] `reasons/cli.py` — The `--model` help text repeats the `ollama:/api:/vertex:` prefix list at ~3300, ~3600, and ~3870; trace which command writes the `costs/*.json` snapshots
- [file] `costs/2026-11-19T180331-update.json` — The persisted shape omits `elapsed_seconds`, suggesting the writer projects a subset of `_record_cost`'s keys
- [general] `provider-tracking-parity` — Should all six provider prefixes produce a `by_model` entry with at least `calls` and `elapsed_seconds`, even when tokens/cost are unknown? Decide the invariant before patching

## Beliefs

- `by-model-entries-created-only-by-record-cost` — `_cost_tracker["by_model"]` keys are created exclusively inside `_record_cost` (`reasons/llm.py:111-117`); no other function inserts into the dict.
- `ollama-bypasses-cli-json-parsing` — `invoke_model` returns ollama stdout before `_parse_cli_json` (`reasons/llm.py:318-319`), and `resolve_model_cmd` emits no `--output-format json` for ollama, so no token data is ever produced.
- `api-and-vertex-skip-cost-recording` — `_invoke_api` returns `response.content` without calling `_record_cost`, so `api:`/`vertex:` calls increment neither global token counters nor `by_model`.
- `elapsed-seconds-update-requires-existing-by-model-entry` — The `finally` block only adds per-model elapsed time when the key already exists (`reasons/llm.py:325-326`), making `sum(by_model[*].elapsed_seconds)` strictly less than the global `elapsed_seconds` for untracked providers.
- `by-model-keys-preserve-provider-prefix` — Recorded keys are the full model string including prefix (e.g. `gemini:<submodel>`, `openai:rits/...`), as seen in `_parse_cli_json`, `_invoke_openai`, and `costs/2026-11-19T180331-update.json`.
- `openai-is-a-recorded-non-cli-provider` — `_invoke_openai` calls `_record_cost(model, input_tokens, output_tokens, 0.0)`, establishing a precedent that non-CLI providers can and should appear in `by_model`.
