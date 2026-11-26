---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `_cost_lock` serializes `_record_cost`, `reset_cost_tracker`, `get_cost_summary`, and the `finally` block in `invoke_model`, and where a race could still surface (e.g. `format_cost_summary` formatting under lock but not copying).

## How `_cost_lock` serializes the cost tracker

`reasons/llm.py` defines a single module-level `_cost_lock = threading.Lock()` and a shared mutable `_cost_tracker` dict. The observed functions all use the same lock:

| Function | Location | Locked work |
|---|---|---|
| `reset_cost_tracker()` | `reasons/llm.py:69` | Resets every top-level field and replaces `by_model` with `{}`. |
| `get_cost_summary()` | `reasons/llm.py:80` | Takes a `copy.deepcopy(_cost_tracker)` while holding the lock, then returns the copy. |
| `format_cost_summary()` | `reasons/llm.py:87` | Binds `s = _cost_tracker` while holding the lock and formats a string from the live dict. |
| `_record_cost()` | `reasons/llm.py:103` | Increments aggregate counters and per-model counters under the lock. |
| `invoke_model()` `finally` | `reasons/llm.py:275`, with the final block around `reasons/llm.py:321-326` | Computes `elapsed` outside the lock, then increments `_cost_tracker["elapsed_seconds"]` and possibly `_cost_tracker["by_model"][model]["elapsed_seconds"]` under the lock. |

Because all internal readers/writers acquire the same non-reentrant lock, these operations are mutually exclusive with respect to each other:

- `_record_cost` cannot interleave two increments and lose an update.
- `reset_cost_tracker` cannot clear fields halfway through a `_record_cost`.
- `get_cost_summary` cannot deepcopy while another internal function is mutating the dict.
- The `finally` block cannot update elapsed time while `get_cost_summary` or `format_cost_summary` holds the lock.

So at the level of individual critical sections, the lock serializes access correctly.

## Where races can still surface

### 1. `_record_cost` and the `finally` elapsed update are not one transaction

A single `invoke_model` call updates cost data in two separate locked sections:

1. `_record_cost(...)` is called from `_parse_cli_json(...)` or `_invoke_openai(...)`.
2. Later, the `finally` block in `invoke_model` updates `elapsed_seconds`.

Those are separate `with _cost_lock:` blocks. A concurrent `get_cost_summary()` or `format_cost_summary()` can run between them and observe:

- `calls`/tokens/cost already incremented,
- `elapsed_seconds` still stale.

Example interleaving:

```text
T1: _record_cost acquires lock, increments calls/tokens/cost, releases
T2: get_cost_summary acquires lock, returns snapshot with new cost but old elapsed
T1: finally acquires lock, increments elapsed_seconds
```

This is not a data race in the Python memory sense, but it is an atomicity/consistency race: there is no atomic “record this call’s cost and elapsed time together” operation.

### 2. `reset_cost_tracker` can interleave between record and elapsed

If `reset_cost_tracker()` runs after `_record_cost` but before the `finally` block, the reset can zero the tracker, and then `finally` still adds the full elapsed time:

```text
T1: _record_cost records cost/calls
T2: reset_cost_tracker clears calls/tokens/cost/elapsed/by_model
T1: finally adds elapsed from the entire invoke_model call
```

Possible results:

- `elapsed_seconds > 0` while `calls == 0`.
- `by_model` may be empty because reset cleared it.
- If another call for the same model records cost after the reset, the older in-flight call’s `finally` may then attach its elapsed time to the new post-reset `by_model[model]` entry because the check is only `if model in _cost_tracker["by_model"]`.

This is one of the main logical races in the current design: reset is serialized with each critical section, but it is not serialized against the full lifecycle of an `invoke_model` call.

### 3. In-flight calls can repopulate elapsed after reset

`invoke_model` captures `t0 = time.monotonic()` before the LLM call and computes `elapsed = time.monotonic() - t0` in `finally`. If `reset_cost_tracker()` runs during the LLM call, the reset clears elapsed, but when the call finishes the `finally` block adds the entire elapsed duration from before the reset.

So a reset during an in-flight call does not fully “reset” timing for that call. The final summary may include time spent before the reset.

### 4. `format_cost_summary` formats under lock but does not copy

`format_cost_summary()` does:

```python
with _cost_lock:
    s = _cost_tracker
    ...
```

This is an alias, not a snapshot. In the current code, that is still safe with respect to the other observed functions because the lock is held for the entire read and formatting operation, and the function returns an immutable string.

However, it has the following limitations:

- It is safe only while every writer obeys `_cost_lock`. If any code mutates `_cost_tracker` directly without the lock, `format_cost_summary` can observe torn/mixed values.
- It would become unsafe if the formatting were moved outside the `with` block while still using `s = _cost_tracker`.
- It would become unsafe if it returned a mutable reference to `_cost_tracker` or nested `by_model` data rather than a string.
- It does not solve the transactional gap between `_record_cost` and `finally`; it only prevents concurrent mutation while the string is being built.

So “formatting under lock but not copying” is not itself the active data race in the observed implementation, but it is a fragile pattern. The safer analogue is `get_cost_summary()`, which deep-copies under the lock because it returns structured data.

### 5. The lock is process-local

`_cost_lock` is a `threading.Lock`, so it only serializes threads inside one Python process. If the application runs multiple processes or multiple server workers, each process has its own `_cost_tracker` and its own lock. Cost summaries are not shared across processes.

### 6. Elapsed time can be attributed to calls that did not record cost

The `finally` block always increments global `elapsed_seconds`, and it increments per-model elapsed if the model key already exists. This means elapsed can be recorded for failed calls, unparsable CLI output, or API paths that did not call `_record_cost` for that invocation. For example, if a previous successful `claude` call created `by_model["claude"]`, a later failed `claude` call still adds its elapsed time to `by_model["claude"]`. The lock serializes the update, but it does not make per-model elapsed proportional to per-model recorded calls.

## Topics to Explore

- [file] `reasons/llm.py` — Contains the full cost tracker, lock, `_record_cost`, `format_cost_summary`, and `invoke_model` lifecycle.
- [function] `reasons/llm.py:_record_cost` — Focus on aggregate and per-model mutation under `_cost_lock`.
- [function] `reasons/llm.py:invoke_model` — Inspect the split between `_record_cost` inside the `try` and elapsed update in `finally`.
- [general] `cost-summary-atomicity` — Consider whether one LLM call should update cost and elapsed in a single locked section or use generation counters/reset epochs.
- [general] `multiprocess-cost-tracking` — Explore how `threading.Lock` behaves across server workers and whether cost stats need shared storage.

## Beliefs

- `cost-lock-serializes-internal-tracker-access` — All observed internal reads/writes of `_cost_tracker` acquire `_cost_lock`.
- `record-cost-and-elapsed-update-are-separate-critical-sections` — `_record_cost` and the `finally` elapsed update cannot be observed atomically together.
- `reset-can-interleave-between-record-and-finally` — `reset_cost_tracker` can clear state between cost recording and elapsed-time recording, producing inconsistent counters.
- `get-cost-summary-returns-deep-copy-under-lock` — `get_cost_summary` deep-copies the tracker while holding the lock, protecting nested `by_model` data.
- `format-cost-summary-uses-live-alias-under-lock` — `format_cost_summary` reads `_cost_tracker` directly under the lock and returns a string, relying on lock discipline rather than copying.
