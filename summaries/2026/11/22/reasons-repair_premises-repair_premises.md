---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: repair_premises in reasons/repair_premises.py

## Overview

`repair_premises` is the top-level orchestrator for the "repair premises" step of a Truth Maintenance System (TMS) pipeline. It takes a batch of premise IDs that were previously flagged as inaccurate by a review step, dispatches each one to `_repair_one` (which calls an LLM to either rewrite or retract the premise), and collects the resulting repair records. It's the batch driver around an inherently-serial per-item LLM operation, with an optional parallel fan-out.

---

## Purpose

After a `review_premises`-style pass marks premises as inaccurate, this function is the fix-it step. It exists as a batch aggregator that:

- Iterates a list of `premise_ids`.
- Calls `_repair_one` per ID (which builds a prompt and invokes an LLM).
- Emits progress to stderr.
- Optionally runs repairs concurrently.
- Optionally invokes a `on_result` callback with the accumulating results so callers can stream progress or persist incrementally.

The intent is that a caller running a review→repair pipeline does not need to reimplement loop/threading/callback plumbing.

---

## Contract

**Preconditions (assumed, not enforced):**

- Every `pid` in `premise_ids` exists as a key in `nodes`. The parallel and sequential branches both call `_repair_one`, which does `nodes[node_id]` (direct indexing) *before* its `try` block — a missing key raises `KeyError` and escapes the function.
- `review_results` may be missing entries for some `pid`s — `_repair_one` uses `review_results.get(node_id, {})`, so missing entries are tolerated.
- `source_contents` may be missing the source path recorded on a node — `_repair_one` uses `.get(source, "(source not available)")`, so a missing source is tolerated with a placeholder.
- `model` and `timeout` are forwarded to `invoke_model`; the caller is responsible for passing something that backend accepts.
- `parallel == 0` means sequential; negative `parallel` is *not* specifically handled — it falls into the sequential branch because of `parallel > 0`.

**Postconditions:**

- Returns a `list[dict]` of the same length as `premise_ids` **if no exception escapes**. Each element is the return of `_repair_one`, which is typically `{"id", "action", "corrected_text", "rationale"}` with `action` in `{"rewrite", "retract", "error"}`.
- **In parallel mode the list is not necessarily in input order** — `as_completed` yields futures in completion order, so `all_results` is order-by-finish, not order-by-`premise_ids`.
- In sequential mode the list **is** in input order.

**Invariants during execution:**

- `on_result` is always invoked with the **same mutable `all_results` list object**, growing by one after each completion.
- `all_results` is only appended to, never reset. Each successful completion appends exactly one element.
- `print(..., file=sys.stderr)` fires once per premise, before or as it completes.

---

## Parameters

| Name | Type / expectation | Notes |
|---|---|---|
| `nodes` | `dict[node_id, node_data]` | Node data from `export_network()`. Values are dicts expected to have `.get('text')`, `.get('source')`. **Assumed to contain every ID in `premise_ids`** — lookup is by `nodes[pid]` in `_repair_one`, not `.get`. |
| `premise_ids` | Iterable of IDs | Falsy (`[]`, `None`) causes an early `return []`. Ordering is preserved in sequential mode only. |
| `source_contents` | `dict[source_path, str]` | Content of source documents. Missing keys fall back to the literal `"(source not available)"` string in the prompt. |
| `review_results` | `dict[node_id, dict]` | Review result per premise; may omit some IDs. `_repair_one` defaults to `{}` if absent. |
| `model` | `str = "claude"` | Forwarded as `model=` to `invoke_model`. |
| `timeout` | `int/float = 300` | Forwarded as `timeout=` to `invoke_model`. |
| `parallel` | `int = 0` | `> 0` → `ThreadPoolExecutor(max_workers=parallel)`. `<= 0` → sequential. Values larger than `len(premise_ids)` are fine (executor caps at submitted tasks). |
| `on_result` | callable or `None` | Called as `on_result(all_results)` after each result. **Receives a live reference** to the shared accumulator, not a copy. Must not mutate it in ways that break subsequent iteration. Must not raise, unless the caller intends to abort the batch. |

---

## Return Value

Returns `list[dict]`. Returns `[]` immediately if `premise_ids` is falsy.

Each element is one of (from `_repair_one`):

- Successful parse: `{"id": <pid>, "action": "rewrite"|"retract", "corrected_text": str|None, "rationale": str}`
- Unparseable but no exception: `{"id": <pid>, "action": "retract", "corrected_text": None, "rationale": "Could not parse repair response"}`
- LLM/parsing raised: `{"id": <pid>, "action": "error", "corrected_text": None, "rationale": "...exception message..."}`

Callers **must** be prepared for:

- `action == "error"` entries (soft failures).
- Non-input ordering when `parallel > 0`. If downstream code joins repairs back to premises by index, it must instead join by `result["id"]`.
- A raised exception (`KeyError`, `on_result` propagating, executor failure) rather than a returned partial list. The function does **not** return partials on raise.

---

## Algorithm

1. **Guard:** `if not premise_ids: return []`. Empty input short-circuits before any I/O or thread pool.
2. **Initialize accumulator:** `all_results = []` — a single list reused throughout.
3. **Branch on `parallel > 0`:**
   - **Parallel branch**
     1. Open `ThreadPoolExecutor(max_workers=parallel)` in a `with` block.
     2. Submit `_repair_one(pid, nodes, source_contents, review_results, model, timeout)` for each `pid`, keyed by `pid` in a `futures` dict.
     3. Loop over `as_completed(futures)` with an index `i` starting at 1. For each completed future:
        - Lookup `pid = futures[future]` (which future was this?).
        - Print `Repaired {i}/{total}: {pid}` to stderr.
        - `future.result()` — either returns the result dict or **re-raises** whatever `_repair_one` raised.
        - Append to `all_results`.
        - If `on_result` is not None, call it with `all_results`.
   - **Sequential branch**
     1. `enumerate(premise_ids, 1)`.
     2. Print `Repairing {i}/{total}: {pid}...`.
     3. Call `_repair_one` directly.
     4. Append and optionally call `on_result`.
4. **Return** `all_results`.

Key subtleties:

- In the parallel branch, `_repair_one` exceptions propagate out of `future.result()` mid-loop. The `with ThreadPoolExecutor(...)` block's `__exit__` will then `shutdown(wait=True)`, *blocking on any not-yet-completed tasks* before the exception escapes. So a single bad input doesn't necessarily mean fast failure — it means "wait for the rest, then raise." Any results that completed before the raise are discarded.
- `on_result` is called with the same growing list; if it snapshots by reference (e.g., stores it as-is), all snapshots alias the same object and later show the final length.
- The parallel branch returns results in completion order. Combined with the shared-list callback, an observer of `on_result` sees completion order, not input order.

---

## Side Effects

- **stderr writes** via `print(..., file=sys.stderr)` once per premise. ASCII progress line. No stdout use.
- **Threads:** when `parallel > 0`, spawns up to `parallel` worker threads. Thread-locals/DB connections/file handles inside `invoke_model` must be thread-safe for this to be correct.
- **LLM calls:** `parallel` (or `1` sequential) invocations of `invoke_model` through `_repair_one`. Each is a network/API hit with `timeout`.
- **Callback invocation:** `on_result` is called `len(all_results)` times total — once per completed item. Caller may use this to persist or display.
- **No mutation** of `nodes`, `source_contents`, or `review_results` — all read-only. `all_results` is internal-only.

---

## Error Handling

- **Soft errors inside `_repair_one`:** wrapped in `try/except Exception` around `invoke_model` + parse. Returns `{"action": "error", ...}`. The batch continues.
- **Parse failures** that don't raise: returns `{"action": "retract", "rationale": "Could not parse repair response"}`. Silent — worth noting since a flaky LLM silently becomes `retract`, which downstream may interpret as "the source doesn't support the claim."
- **`KeyError` from `nodes[node_id]`:** happens *before* `_repair_one`'s `try`. This propagates out of `_repair_one`, then out of `future.result()` (parallel) or out of the direct call (sequential), and terminates `repair_premises`. Partial results are **not** returned or delivered to `on_result` beyond those already appended before the raise.
- **`on_result` raising:** propagates the same way. In sequential mode, aborts remaining work; in parallel mode, the `with` block joins pending future tasks before re-raising.
- **`ThreadPoolExecutor` errors** (e.g., `RuntimeError` on shutdown from an interpreter teardown path): not specially handled.
- **`Accumulator mutation by callback:** not detected. If `on_result` clears `all_results`, all subsequent state is corrupted.

---

## Usage Patterns

Typical call shape in a review→repair pipeline:

```python
network = export_network(...)
review = review_premises(...)          # produces review_results
inaccurate_ids = [pid for pid, r in review.items() if r.get("action") == "rewrite"]
repairs = repair_premises(
    nodes=network["nodes"],
    premise_ids=inaccurate_ids,
    source_contents=load_sources(...),
    review_results=review,
    model="claude",
    parallel=8,
    on_result=lambda rs: persist_incrementally(rs),   # receives live list
)
# Persist/apply repairs. Join by result["id"], NOT by index, if parallel>0.
```

Caller obligations:

1. **Guarantee `premise_ids ⊆ nodes`** to avoid `KeyError`.
2. **Join by `id`**, not by position, when `parallel > 0`.
3. **Handle `action == "error"`** rows explicitly — they signal an LLM/parse failure and do not represent a decision.
4. If using `on_result` to persist partial progress, **do not assume it is the full list at that moment** — take a snapshot (`list(rs)`) if you need one.
5. Do not mutate the list passed to `on_result` — it's the internal accumulator.
6. Provide `invoke_model`-thread-safe dependencies if `parallel > 0`.

---

## Dependencies

- `concurrent.futures.ThreadPoolExecutor`, `as_completed` — parallel fan-out.
- `json` — used indirectly in `_repair_one` via `parse_repair_response`.
- `sys` — stderr progress.
- `reasons.llm.invoke_model` — the actual LLM call.
- `format_premise_for_repair`, `parse_repair_response`, `_repair_one`, `REPAIR_PREMISES_PROMPT` — module-internal helpers. Of these, `_repair_one` is the only direct call from this function; the rest are reached through it.

Assumptions not enforced by the type system:

- `nodes[pid]` values are dicts supporting `.get("source")` and `.get("text")`.
- `source_contents` values are strings suitable for embedding in the prompt as fenced code (no backtick sanitization is performed — a source containing ` ``` ` would prematurely close the fence in the prompt).
- The LLM backend can be called concurrently (thread-safe client, thread-safe session, or independent client per call).
- LLM output is JSON or quasi-JSON parseable by `parse_repair_response`.

---

## Topics to Explore

- [function] `reasons/repair_premises.py:_repair_one` — The per-premise operation this batch wraps; contains the prompt construction, LLM invocation, and soft-error fallbacks that define the shapes of returned dicts.
- [function] `reasons/repair_premises.py:parse_repair_response` — Explains why some LLM outputs become `retract` silently; worth knowing its scanning strategy and what it rejects.
- [file] `reasons/llm.py` — `invoke_model` is where timeout, model routing, retries, and thread-safety semantics live; necessary to judge whether `parallel > 0` is safe.
- [file] `tests/test_repair_premises.py` — Shows the observed ordering, callback-throttling, and error-case behavior that downstream code has come to rely on.
- [general] `tms-review-to-repair-pipeline` — Understand where `review_results` come from and how `action == "error"` or `retract` should be handled by the caller (retraction semantics in the TMS).

---

## Beliefs

- `parallel-results-not-in-input-order` — When `parallel > 0`, `all_results` is ordered by completion, not by `premise_ids`.
- `on-result-receives-shared-mutable-list` — `on_result` is called with the same `all_results` reference each time; callers must copy if they want a snapshot.
- `missing-node-id-raises-key-error` — A `premise_id` absent from `nodes` raises `KeyError` because `_repair_one` uses `nodes[node_id]` before its `try` block.
- `_repair_one-swallows-llm-errors` — LLM and parse failures inside `_repair_one` are converted to dicts with `action == "error"` or `"retract"` rather than raised.
- `empty-premise-ids-returns-empty-list` — A falsy `premise_ids` short-circuits to `[]` before any I/O or thread-pool construction.
- `parallel-requires-thread-safe-invoke-model` — Correctness with `parallel > 0` depends on `invoke_model` and any global state it touches being thread-safe.
