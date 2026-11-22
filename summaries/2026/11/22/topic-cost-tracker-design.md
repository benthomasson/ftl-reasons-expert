---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The global-vs-per-model asymmetry (elapsed always, tokens/cost conditionally, per-model elapsed only after a `_record_cost`) is a subtle design choice worth designing against for any new backend.

# The Global-vs-Per-Model Asymmetry in `reasons/llm.py`

## The Setup

`reasons/llm.py` maintains a single process-global tracker (`_cost_tracker`, defined around line 60) with two scopes: a flat global bundle and a `by_model` dict. Three different counters accumulate under three **different** rules, and that's where the subtlety lives.

### Rule 1 — Global `elapsed_seconds`: always

In `invoke_model`'s `finally` block (lines 322–326):

```python
finally:
    elapsed = time.monotonic() - t0
    with _cost_lock:
        _cost_tracker["elapsed_seconds"] += elapsed
        if model in _cost_tracker["by_model"]:
            _cost_tracker["by_model"][model]["elapsed_seconds"] += elapsed
```

Line 324 runs unconditionally on **every** call to `invoke_model` — success, failure, `FileNotFoundError`, `subprocess.TimeoutExpired`, `ImportError` for `api:`/`vertex:`, whatever. It doesn't matter whether the backend reported tokens, or whether the call did any work at all.

### Rule 2 — Global `calls` / `input_tokens` / `output_tokens` / `total_cost_usd`: only via `_record_cost`

`_record_cost` (line 103) is the *only* place that increments the global call/token/cost counters. It's invoked from exactly three call sites:

- `_parse_cli_json` → Gemini branch (line 145)
- `_parse_cli_json` → Claude/default branch (line 153)
- `_invoke_openai` (line 271)

Notably **not** called from `_invoke_api` (the `api:`/`vertex:` LangChain path) and **not** called for `ollama:` calls (which return raw stdout without a JSON parse). It's also not called if `_parse_cli_json` fails to parse (`test_invalid_json_returns_raw`, `test_json_array_returns_raw` in `tests/test_llm.py` both assert `calls == 0`).

### Rule 3 — Per-model `elapsed_seconds`: only after `_record_cost` has created the entry

Line 326 is guarded by `if model in _cost_tracker["by_model"]`. The only place a `by_model` entry can be created is inside `_record_cost` (lines 110–114):

```python
if model not in _cost_tracker["by_model"]:
    _cost_tracker["by_model"][model] = {
        "calls": 0, "input_tokens": 0, "output_tokens": 0,
        "total_cost_usd": 0.0, "elapsed_seconds": 0.0,
    }
```

So `elapsed_seconds` for a model is silently dropped until a *successful, parseable* `_record_cost` has run first. The `finally` block was written to "not throw if the model isn't there yet" — but the consequence is that *some* backends never contribute per-model time at all.

## What the Asymmetry Actually Produces

The three rules compose into this table across the four backends reachable through `invoke_model`:

| Backend | Path | Global elapsed | Global calls/tokens/cost | Per-model elapsed |
|---|---|---|---|---|
| `claude`, `gemini`, `claude:<m>`, `gemini:<m>` | subprocess → `_parse_cli_json` → `_record_cost` | ✅ | ✅ if JSON valid | ✅ |
| `openai:<m>` | `_invoke_openai` → `_record_cost` | ✅ | ✅ (0 cost, real tokens) | ✅ |
| `api:<m>`, `vertex:<m>` | `_invoke_api` (LangChain, no cost capture) | ✅ | ❌ | ❌ |
| `ollama:<m>` | subprocess, raw stdout, no parse | ✅ | ❌ | ❌ |

Three second-order consequences:

1. **Failed calls pollute global elapsed but not global calls.** A run of ten `claude` calls where nine time out still adds ~9× the timeout to `_cost_tracker["elapsed_seconds"]`, but the calls counter only reflects the one successful call. `format_cost_summary` (line 88) gates on `if s["calls"] == 0: return ""`, so a trajectory of pure failures shows nothing — but a *mixed* trajectory shows elapsed that over-reports relative to `calls`/`tokens`/`cost`.

2. **`api:`/`vertex:`/`ollama:` are invisible in `by_model`.** A `get_cost_summary()` after a pure `api:` workload returns `by_model == {}` even though global `elapsed_seconds > 0` — contradicting the intuitive reading of a "per-model summary."

3. **The tests don't cover the negative path.** `tests/test_llm.py:427–447` exercise `_record_cost` directly (so they always create the entry), and `test_invokes_subprocess` (line ~48) goes through the `claude` path where `_parse_cli_json` fires. There is *no* test asserting that `by_model["ollama:gemma3:4b"]["elapsed_seconds"]` stays at `0.0` — nor that `api:` calls produce a `by_model` key at all. The asymmetry is untested and therefore easy to regress.

## Why It Happens (and Why It's "Deliberate-Adjacent")

The asymmetry is defensible in two of its three pieces:

- **Elapsed unconditional** makes sense: wall-clock time is *always* observable, even when usage isn't. You don't need to fabricate anything.
- **Tokens/cost conditional on `_record_cost`** makes sense: token counts can only be reported by backends that *report* them; counting a call as `+0 tokens` would be lying.
- **Per-model elapsed conditional on `by_model` existence** is the accidental piece. It's a consequence of `_record_cost` doing double duty as both "record usage" *and* "lazy-initialize the per-model slot." The `finally` block was written to be defensive, but the cost is that the per-model elapsed is coupled to whether the *usage* path ran, rather than to whether the *call* ran.

## Designing Against It for a New Backend

If you add a new backend (say `together:<m>` or a new local runner), you have four choices, and picking the wrong one silently bakes in a gap:

1. **Do nothing** — your backend inherits the `api:`/`ollama:` profile: global elapsed only, absent from `by_model`, no calls/tokens/cost. Fine only if you intend the backend to be "unmetered but timed."

2. **Always call `_record_cost(model, …)` with the real numbers** — this is the `openai:` pattern (`_invoke_openai`, line 271). Cost can be `0.0` if you don't have pricing, but tokens should be real. This is the *minimal* way to go from "invisible" to "visible."

3. **Decouple initialization from recording.** The cleanest fix is to break `_record_cost` into `_ensure_model_entry(model)` + `_add_usage(...)`, and call `_ensure_model_entry` unconditionally from `invoke_model`'s `try` or the top of the `finally`. Then `api:`/`vertex:`/`ollama:` would show up in `by_model` with `calls=0, tokens=0, cost=0, elapsed>0` — an honest summary of "we spent time, we don't know what it cost."

4. **Introduce `_record_elapsed(model, elapsed)`** symmetric to `_record_cost`, and call it from the `finally` regardless of whether usage was reported. This keeps `_record_cost`'s contract unchanged ("only call when usage is known") while making the per-model time path backend-agnostic.

The subtle trap is (1): it *looks* free because `invoke_model`'s signature doesn't change and nothing raises. But `format_cost_summary` (line 88–100) and `get_cost_summary` read `_cost_tracker` directly, and consumers in `reasons/cli.py` (lines 1765, 2171, 2206, 2279) only print — they don't reconcile `elapsed_seconds` against `calls`. So the asymmetry ships silently.

### Reader Takeaway

Any new backend needs an explicit answer to two questions that the current code conflates:

- **Does this backend report usage?** → determines whether `_record_cost` fires.
- **Should this backend appear in `by_model`?** → should be independent, but currently isn't.

If those two answers ever diverge (a backend you want to time per-model but can't meter), you must either change `invoke_model`'s `finally` block (lines 322–326) or accept that `by_model` will be empty for that backend. Add a test that asserts the negative case before shipping.

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — The `try`/`finally` at lines 322–326 is the single place where the three accumulation rules interleave; the conditional on line 325 is the crux of the asymmetry.
- [function] `reasons/llm.py:_record_cost` — Doubles as both usage-recorder and lazy per-model initializer (lines 105–114); splitting those responsibilities is the fix for the per-model elapsed gap.
- [function] `reasons/llm.py:_invoke_api` — The `api:`/`vertex:` path that produces global elapsed without any `_record_cost` call; a template for "backend with no usage data."
- [file] `tests/test_llm.py` — Look specifically at `TestCostTracker` (lines ~420–469) and `TestInvokeModel` around `test_ollama_command`: no test asserts `by_model` is empty for `ollama`/`api:` after a call.
- [general] `cost-tracker-symmetry` — Cross-backend invariant: which counters should exist for every backend regardless of whether it reports usage, and how to enforce that in code rather than by convention.

## Beliefs

- `elapsed-seconds-requires-no-usage-reporting` — `_cost_tracker["elapsed_seconds"]` is incremented in `invoke_model`'s `finally` (line 324) even when `_record_cost` is never called, so `api:`/`vertex:`/`ollama:` calls contribute to global elapsed but not to global calls/tokens/cost.
- `by-model-entry-requires-record-cost` — A `_cost_tracker["by_model"]` entry can only be created inside `_record_cost` (lines 110–114); consequently the per-model elapsed increment at line 326 is skipped for any backend that never invokes `_record_cost`.
- `api-and-vertex-skip-cost-recording` — `_invoke_api` returns only `response.content` and never calls `_record_cost`, so `api:` and `vertex:` models never appear in `get_cost_summary()["by_model"]`.
- `ollama-skips-json-parsing` — `invoke_model` returns raw `output` for `ollama:` models before reaching `_parse_cli_json` (near line 315), so Ollama calls contribute elapsed only, never tokens or a `by_model` slot.
- `format-gated-on-calls-not-elapsed` — `format_cost_summary` returns `""` when `calls == 0` even if `elapsed_seconds > 0`, so a pure-failure run reports nothing despite having accumulated wall-clock time.
- `failed-calls-inflate-global-elapsed` — Because the `finally` at lines 322–326 runs on exceptions (e.g., `FileNotFoundError`, `subprocess.TimeoutExpired`), failed invocations add to global `elapsed_seconds` while leaving `calls` untouched.
