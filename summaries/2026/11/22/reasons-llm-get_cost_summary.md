---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: get_cost_summary in reasons/llm.py

## `get_cost_summary` — `reasons/llm.py`

### 1. Purpose

Returns a **snapshot** of the module's cumulative LLM usage telemetry — call counts, input/output token totals, dollar cost, elapsed wall-clock time, and a per-model breakdown. It exists because the tracker (`_cost_tracker`) is mutated from multiple places (the `finally` block of `invoke_model`, `_record_cost` called from `_parse_cli_json` and `_invoke_openai`, and `reset_cost_tracker`), and callers need a **safe, isolated** view they can inspect, serialize, or log without racing other threads and without accidentally mutating the tracker.

### 2. Contract

**Preconditions**
- The module has been imported (so `_cost_tracker` and `_cost_lock` exist).
- No requirement that any LLM calls have happened; a fresh/reset tracker is valid input.

**Postconditions**
- Returns a `dict` with the same shape as `_cost_tracker` at the instant the lock was held.
- The returned dict is **fully decoupled** from `_cost_tracker`: mutating it (including nested `by_model` entries) has no effect on the global tracker, and vice versa.
- The lock is released before the function returns to the caller (the `return` inside `with` triggers `__exit__`).

**Invariants**
- `_cost_tracker` shape is stable: keys `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, `elapsed_seconds`, `by_model`, with `by_model` mapping `model_name -> {calls, input_tokens, output_tokens, total_cost_usd, elapsed_seconds}`. `get_cost_summary` does not add or remove keys — it echoes whatever `reset_cost_tracker` and `_record_cost` have produced.
- Reading the summary is **non-destructive**: it does **not** reset the tracker.

### 3. Parameters

None. The signature is `get_cost_summary() -> dict`.

### 4. Return Value

Always a `dict` (never `None`, never raises under normal conditions). Structure:

| key | type | meaning |
|---|---|---|
| `calls` | `int` | total LLM calls recorded |
| `input_tokens` | `int` | summed input tokens across all calls |
| `output_tokens` | `int` | summed output tokens |
| `total_cost_usd` | `float` | summed reported cost (often `0.0` for providers that don't report cost) |
| `elapsed_seconds` | `float` | summed wall-clock time across `invoke_model` invocations |
| `by_model` | `dict[str, dict]` | per-model breakdown, lazily populated |

**Caller obligations**
- Treat the result as opaque telemetry; do not assume `by_model` contains a specific model unless that model was actually invoked since the last reset.
- Cost fields may be `0.0` even when calls occurred (Gemini CLI and OpenAI Responses API paths record `0.0` cost); token fields may be `0` if the provider's JSON did not expose usage. Callers must not interpret `total_cost_usd == 0.0` as "no calls".

### 5. Algorithm

1. `import copy` — deferred import of the stdlib `copy` module. This keeps `copy` out of the module's top-level namespace; it is functionally equivalent to a top-level import.
2. `with _cost_lock:` — acquire the reentrant-safe `threading.Lock` guarding all mutations of `_cost_tracker` (`_record_cost`, `reset_cost_tracker`, and the `finally` block in `invoke_model`). This blocks until no writer is active and prevents a writer from starting mid-copy.
3. `return copy.deepcopy(_cost_tracker)` — perform a **deep** copy while holding the lock. Deep copy (rather than `dict(...)` or a shallow copy) is required because `_cost_tracker["by_model"]` is a nested `dict[str, dict]`; a shallow copy would share the inner dicts and let callers corrupt the tracker by mutating a returned `by_model[m]`.
4. Context manager `__exit__` releases the lock as the `return` unwinds.

### 6. Side Effects

- Acquires and releases `_cost_lock` (blocking if a writer is active).
- No mutation of `_cost_tracker`, no I/O, no logging, no global state changes.
- No network calls — this is purely in-memory snapshotting, unlike `format_cost_summary`'s sibling functions.

### 7. Error Handling

- The function itself has no explicit `try/except`.
- **Deadlock risk (not a raised error):** as with all `with _cost_lock:` blocks, if some other code path acquires `_cost_lock` and never releases it, `get_cost_summary` will block indefinitely. Inspection of the file shows all acquisitions use `with`, so this should not occur.
- `copy.deepcopy` on the tracker's contents (ints, floats, plain dicts) cannot fail in practice, but if the tracker were ever extended with a non-copyable object (e.g. a socket, thread lock, or generator), `deepcopy` would raise `TypeError`. This is an implicit assumption not enforced by the type system (`_cost_tracker` is annotated only as the loose type `dict`).
- If called after `reset_cost_tracker`, it returns a valid zeroed dict — it does not raise or return a sentinel.

### 8. Usage Patterns

Typical flow:
```python
reset_cost_tracker()
result = invoke_model(prompt, model="claude")
summary = get_cost_summary()
log.info("calls=%d tokens_in=%d tokens_out=%d cost=$%.4f",
         summary["calls"], summary["input_tokens"],
         summary["output_tokens"], summary["total_cost_usd"])
```
- **Snapshot, don't hold it:** Since the returned dict is a deep copy, callers can safely pass it into serializers (JSON, logging, test assertions) without re-acquiring the lock.
- **Do not use it as a live view:** It is a point-in-time snapshot. Subsequent `invoke_model` calls will not update it — call again if you need fresher data.
- **Tests** in `tests/test_llm.py` likely pair `reset_cost_tracker()` (setup) with `get_cost_summary()` (assertion) and rely on the deep-copy behavior to avoid cross-test contamination.

### 9. Dependencies

- `threading` — provides `_cost_lock` (module-scope, single global lock shared with `_record_cost`, `reset_cost_tracker`, and the `finally` block of `invoke_model`).
- `copy` — stdlib, imported locally.
- Global `_cost_tracker` and `_cost_lock` in `reasons/llm.py`. There are no external frameworks (LangChain, Langfuse, OpenAI) involved; this function works even if all optional provider imports failed.

### Assumptions not enforced by the type system

- The `-> dict` annotation is unparameterized. Callers get **no static checking** on key names or value types; typos like `summary["call_count"]` will surface as `KeyError` at runtime.
- That `_cost_tracker`'s values are deep-copyable (currently true; would silently break the concurrency guarantee if extended with non-copyable objects).
- That `by_model` keys are model strings passed to `_record_cost` — but `_record_cost` is called from multiple code paths with slightly different string shapes (e.g. `"claude"`, `"claude:sonnet"`, `"openai:gpt-4o"`), so callers cannot assume canonicalization.
- That elapsed_seconds is fully attributed at the time of the snapshot: the `finally` in `invoke_model` updates elapsed under `_cost_lock`, so a call currently in flight is neither in the summary nor half-recorded — but a long-running call will make the summary appear stale relative to wall clock until it returns.

## Topics to Explore

- [function] `reasons/llm.py:_record_cost` — Populates `_cost_tracker` and lazily creates `by_model` entries; the counterpart mutation site that `get_cost_summary` snapshots.
- [function] `reasons/llm.py:reset_cost_tracker` — Mutates the tracker in place instead of replacing it; explains why identity-stable deep copies matter.
- [function] `reasons/llm.py:format_cost_summary` — Human-readable rendering that reads `_cost_tracker` directly under the lock instead of going through `get_cost_summary`; useful to compare snapshot vs. direct read.
- [file] `tests/test_llm.py` — Shows the concrete setup/assertion pattern for cost tracking, including which providers emit nonzero cost/token values.
- [general] `thread-safety-of-_cost_lock` — How `_cost_lock` serializes `_record_cost`, `reset_cost_tracker`, `get_cost_summary`, and the `finally` block in `invoke_model`, and where a race could still surface (e.g. `format_cost_summary` formatting under lock but not copying).

## Beliefs

- `get-cost-summary-returns-deep-copy` — `get_cost_summary` returns `copy.deepcopy(_cost_tracker)`, so mutations to the returned dict never leak into the global tracker.
- `cost-tracker-access-is-lock-serialized` — Every read or write of `_cost_tracker` in this module is performed while holding `_cost_lock`.
- `get-cost-summary-is-non-destructive` — Calling `get_cost_summary` does not reset or modify accumulated counters.
- `by-model-is-lazily-populated` — `_cost_tracker["by_model"]` starts empty and only gains entries when `_record_cost` is first called for a given model string.
- `cost-summary-may-underreport-cost` — Because several provider paths (`_invoke_openai`, Gemini CLI parsing) call `_record_cost` with `cost_usd=0.0`, `total_cost_usd` can be `0.0` even when `calls > 0`.
