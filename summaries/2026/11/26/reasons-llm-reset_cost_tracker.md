---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: reset_cost_tracker in reasons/llm.py

## `reset_cost_tracker`

### Purpose

Resets the process-wide, module-level `_cost_tracker` dictionary back to its initial state so that accumulated LLM cost/token statistics start from zero. It exists so callers (test suites, long-running processes, or CLI entry points that want a clean accounting baseline per run) can clear previously-accumulated metrics without re-importing the module or rebuilding the tracker by hand.

It is the paired "clear" operation to `_record_cost`'s "accumulate" and `get_cost_summary`'s "read".

### Contract

- **Preconditions:** None. It is safe to call at any time, including on a freshly imported module, and — because it takes the lock — safe to call while other threads may be calling `_record_cost`, `_parse_cli_json`, or `invoke_model` concurrently.
- **Postconditions:** After the lock is released:
  - `_cost_tracker["calls"] == 0`
  - `_cost_tracker["input_tokens"] == 0`
  - `_cost_tracker["output_tokens"] == 0`
  - `_cost_tracker["total_cost_usd"] == 0.0`
  - `_cost_tracker["elapsed_seconds"] == 0.0`
  - `_cost_tracker["by_model"] == {}`
- **Invariants it must preserve:**
  - The set of keys in `_cost_tracker` is **not** changed — the same six keys stay present. Callers such as `get_cost_summary` and `format_cost_summary` assume those keys exist ("calls", "input_tokens", "output_tokens", "total_cost_usd", "elapsed_seconds", "by_model").
  - The dict object identity is preserved (`_cost_tracker` is mutated in place; the name is never rebound). This matters if any code elsewhere holds a reference to the dict (though the module never does today; all reads go through the lock and `copy.deepcopy`).
  - All mutations happen while holding `_cost_lock`, so no read via `get_cost_summary`/`format_cost_summary` and no concurrent `_record_cost` can observe a partially-reset tracker.

### Parameters

None. It takes no arguments and has no `*args`/`**kwargs`.

### Return Value

Returns `None`. There is no success/failure signal. Callers cannot tell whether a reset occurred beyond trusting the postcondition.

### Algorithm

Step-by-step:

1. Acquire `_cost_lock` (a module-level `threading.Lock()`) via `with _cost_lock:`. This blocks until other writers/readers that also hold the lock release it.
2. Inside the lock, reassign each scalar key in `_cost_tracker` to its zero value:
   - `"calls"` → `0` (int)
   - `"input_tokens"` → `0` (int)
   - `"output_tokens"` → `0` (int)
   - `"total_cost_usd"` → `0.0` (float)
   - `"elapsed_seconds"` → `0.0` (float)
3. Rebind `"by_model"` to a **fresh, empty dict** `{}` rather than calling `.clear()` on the existing one. This drops the previous per-model dict entries and gives a new object with no external aliases.
4. Release the lock when the `with` block exits, and return `None`.

Note the mixed technique: scalars are re-assigned (overwritten in place on the same outer dict), whereas `by_model` is replaced with a new dict object. Both are safe because nothing else in the module keeps a long-lived reference to `by_model`.

### Side Effects

- **State mutation:** Zeroes out the module-global `_cost_tracker`, which is read by `get_cost_summary()`, `format_cost_summary()`, and written by `_record_cost()` (and indirectly by `_parse_cli_json`, `_invoke_openai`, and `invoke_model`'s `finally` block).
- **Lock acquisition:** Briefly holds `_cost_lock`, which can serialize with concurrent LLM calls being recorded.
- **No I/O**, no prints, no retries, no return value.

### Error Handling

- No exceptions are raised in normal operation. All operations are in-memory dict assignments.
- **Theoretical risks:**
  - If `_cost_tracker` were replaced by a non-dict (impossible given the module-level literal), this function would raise `TypeError`. Not reachable.
  - `threading.Lock` acquisition has no timeout; if another thread that holds `_cost_lock` were to block forever (e.g., a hung `_record_cost` — impossible in practice since it's a short critical section), this function would block. There is no deadlock path in the current code because every lock holder releases it synchronously.
- No reset is attempted for the `_langfuse_handler` cache or anything else — only `_cost_tracker` is affected.

### Usage Patterns

- Typically called once at the start of a test (e.g., in a fixture in `tests/test_llm.py`) or at the start of a run/session where callers want per-run cost accounting.
- Callers who want the previous totals before resetting must snapshot with `get_cost_summary()` **before** calling this function, since it discards data. `get_cost_summary()` returns a `deepcopy`, so a snapshot taken beforehand is safe.
- It is **thread-safe but not transactional**: if a concurrent `invoke_model` is midway through an LLM call, the `finally` block's `elapsed_seconds` accounting will still execute afterward and add elapsed time to the fresh tracker. So a reset racing an in-flight call will not necessarily leave `elapsed_seconds == 0.0` at the end — the pre-reset call's elapsed time lands in the post-reset tracker. This "leak" is by design (any reset semantics for in-flight work would require cooperation from `invoke_model`), but it is an assumption the type system does not enforce and which a caller may not realize.
- Because it zeroes `elapsed_seconds` but does not touch `_langfuse_checked`/`_langfuse_handler`, resets do not affect Langfuse tracing initialization.

### Dependencies

- `threading` (standard library) — for `_cost_lock`.
- The module-level `_cost_lock` and `_cost_tracker` objects defined near the top of `reasons/llm.py`. No external packages.

---

## Topics to Explore

- [function] `reasons/llm.py:get_cost_summary` — How reads are isolated via deepcopy under the same lock; useful contrast to `reset_cost_tracker`'s in-place mutation.
- [function] `reasons/llm.py:_record_cost` — The paired accumulate operation that populates the tracker this function clears, including per-model `elapsed_seconds` updates.
- [function] `reasons/llm.py:invoke_model` — Its `finally` block mutates the tracker after the call, which is why a reset can race with an in-flight call.
- [file] `tests/test_llm.py` — Where `reset_cost_tracker` is likely invoked as a fixture; shows expected usage and any assumptions tests encode.
- [general] `thread-safety-of-shared-counters` — Why a single `threading.Lock` around a mutable module dict is used here, and what invariants that protects.

## Beliefs

- `reset_cost_tracker-requires-cost-lock` — `reset_cost_tracker` requires `_cost_lock` and acquires it for every mutation, which is the only synchronization protecting `_cost_tracker`.
- `reset-cost-tracker-preserves-keys` — `reset_cost_tracker` preserves the six keys of `_cost_tracker` and only replaces their values; it never adds or removes keys.
- `reset-cost-tracker-replaces-by-model-dict` — `reset_cost_tracker` re-binds `_cost_tracker["by_model"]` to a new empty dict rather than clearing the existing one in place.
- `reset-cost-tracker-produces-empty-tracker` — After `reset_cost_tracker` returns (absent concurrent in-flight calls), `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`, and `elapsed_seconds` are all zero and `by_model` is empty.
- `reset-cost-tracker-causes-loss-of-accumulated-stats` — Callers that need previous totals must snapshot via `get_cost_summary()` before calling `reset_cost_tracker`, since the reset discards all accumulated data.
