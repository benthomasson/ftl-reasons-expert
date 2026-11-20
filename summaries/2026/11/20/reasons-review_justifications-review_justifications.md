---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: review_justifications in reasons/review_justifications.py

# `review_justifications` — Explanation

## 1. Purpose

`review_justifications` audits the **justification mode** of a subset of beliefs in a Truth Maintenance System (TMS). Every belief in the network that was derived via a **SL (Support-List) justification** may currently be marked as *ALL* (conjunctive — every antecedent must be IN) or *ANY* (disjunctive — any one antecedent suffices). Many systems default to ALL when they should be ANY, and vice versa.

This function:

- Selects beliefs that are **IN** and have at least one SL justification with enough antecedents to be interesting (`min_antecedents`, default 2).
- Chunks those beliefs into **batches** and asks an LLM to classify each one as `ALL` / `ANY` / `MIXED`, optionally in parallel.
- Returns the list of LLM classifications for human review.

Crucially, it is **advisory only** — the module docstring says "Reports candidates for conversion but does not modify the database." It never mutates `nodes`.

## 2. Contract

**Preconditions (unenforced):**
- `nodes` is a `dict[str, dict]` mapping node IDs to node data. Each node is assumed to have `truth_value` and `justifications`, where `justifications` is a list of dicts with `type` (e.g. `"SL"`) and `antecedents` (list of node IDs).
- `belief_ids`, if provided, is a list of node IDs present in `nodes` — but IDs that fail the filter are silently dropped, not error.
- `on_batch`, if provided, is callable accepting a list.
- `_process_batch`, `invoke_model`, and `format_belief_for_review` work as documented.

**Postconditions:**
- Returns a (possibly empty) list of dicts, each `{id, classification, required_antecedents, independent_antecedents, comment}` as produced by `parse_justification_review`.
- Never raises on per-batch LLM failures — those are logged and skipped.
- `nodes` is not mutated.

**Invariants:**
- Only nodes with `truth_value == "IN"` are ever sent to the LLM.
- Only nodes where **at least one** SL justification has `len(antecedents) >= min_antecedents` are considered.
- Serial mode preserves input order; parallel mode does **not**.

## 3. Parameters

| Param | Meaning | Edge cases |
|---|---|---|
| `nodes` | Mapping of node_id → node data. Source of truth for evidence. | If a node lacks `justifications`, `_has_multi_antecedent_sl` treats it as non-candidate. |
| `belief_ids` | Optional explicit list of IDs to review instead of scanning all of `nodes`. | Still re-filtered — non-IN or insufficient-antecedent IDs are dropped silently. This claim of "only review these IDs" is weak. |
| `model` | LLM model string forwarded to `invoke_model`. Default `"claude"`. | No validation; an invalid model raises inside `_process_batch` → caught per batch → warn + skip. |
| `timeout` | Per-LLM-call timeout in seconds. Default 300. | `< 0` or `None` semantics depend on `invoke_model`. |
| `batch_size` | Beliefs per LLM call. Default `REVIEW_BATCH_SIZE = 15`. | **Must be > 0**; if `0`, `range(0, n, 0)` raises `ValueError` in the batching loop — this is one of the few exceptions that can escape. Negative values also behave unexpectedly. |
| `min_antecedents` | Minimum number of antecedents in an SL justification for the belief to be considered. Default 2. | `<= 1` effectively includes single-antecedent justifications (which are usually trivially ALL). |
| `on_batch` | Optional callback invoked after each successful batch with the **cumulative** `all_results` list. | Not called when zero candidates. In both branches it is *inside* the `try`, so exceptions from it are caught and mislabeled as "batch failed". |
| `parallel` | If `> 0`, process batches via `ThreadPoolExecutor(max_workers=parallel)`. | Greater than the number of batches is harmless (extra workers just unused). `0` → serial. |

## 4. Return Value

A `list[dict]` where each dict has:
- `id`: the belief being classified.
- `classification`: one of `"ALL"`, `"ANY"`, `"MIXED"` (uppercased; defaults to `"ALL"` if the LLM response's `classification` is not a string).
- `required_antecedents`: list of IDs (empty for ANY).
- `independent_antecedents`: list of IDs (empty for ALL).
- `comment`: free-text string; empty string if missing.

Returned results **only reflect beliefs for which the LLM produced parseable output**. Failed batches contribute nothing (no error entry). Callers must treat the result as a **partial** set keyed by `id`, not a dense answer per input ID.

Empty return happens when:
- No beliefs match the filter, **or**
- Every batch raised or returned unparseable text.

## 5. Algorithm

```
1. If belief_ids is None:
       belief_ids = sorted ids where truth_value == "IN"
                                 and _has_multi_antecedent_sl(node, min_antecedents)
   Else:
       belief_ids = [id for id in belief_ids
                     if id in nodes
                     and truth_value == "IN"
                     and _has_multi_antecedent_sl(node, min_antecedents)]
2. If empty → return [].
3. Chunk belief_ids into batches of size batch_size.
4. If parallel > 0:
       Pool of `parallel` threads.
       Submit _process_batch(batch, nodes, model, timeout) for each batch.
       As each future completes (out-of-order):
           extend all_results; log "Batch ... complete";
           call on_batch(all_results) if provided.
       Exceptions in a future → warn to stderr; that batch's results lost.
   Else (serial):
       For each batch, log "Reviewing batch ...", call _process_batch synchronously,
       extend all_results, call on_batch if provided.
       Exceptions → warn to stderr; continue.
5. Return all_results.
```

`_process_batch` builds prompt text by joining `format_belief_for_review(nid, nodes)` for each ID, invokes the LLM, then parses with `parse_justification_review` (which scans for the first `[...]` JSON array and skips malformed entries).

## 6. Side Effects

- **Network I/O**: one LLM call per batch (via `invoke_model`).
- **stderr logging**: progress messages ("Reviewing batch N/M ...", "Batch N/M complete ...", and `WARN:` on failure).
- **Callback invocation**: `on_batch(all_results)` is called after each batch. `all_results` is the **same mutable list** throughout — callers who store it must copy.
- **No mutation** of `nodes` or any database.
- **Threads**: with `parallel > 0`, concurrent worker threads perform LLM calls against the same `nodes` dict (read-only).

## 7. Error Handling

- **Per-batch exceptions** from `_process_batch` (e.g. `invoke_model` timeout, malformed response, network error) are caught and logged as `WARN`. The batch is dropped.
- **`on_batch` exceptions** are also caught (both branches) and misleadingly logged as `batch N failed`. Partial results already extended into `all_results` before `on_batch` raised **are still kept**.
- **`batch_size <= 0`** raises `ValueError` from `range(0, n, 0)` — this is **not** caught and will propagate.
- **`nodes` being `None`** raises `AttributeError` immediately (`nodes.items()`).
- **Individual entries inside results** are already normalized by `parse_justification_review`, so a bad LLM entry silently degrades to `classification="ALL"`, empty antecedent lists.
- Nothing is raised on "all batches failed" — returns `[]`.

## 8. Usage Patterns

Typical call from a CLI/pipeline after `export_network()`:

```python
results = review_justifications(nodes, model="claude", parallel=4)
for r in results:
    if r["classification"] != "ALL":
        print(f"{r['id']} → {r['classification']}: {r['comment']}")
```

Caller obligations:

- Treat the return as **non-exhaustive**; do not assume every input ID is present.
- Do not retain `on_batch`'s argument — it mutates.
- Do not rely on ordering when `parallel > 0`.
- To review specific IDs, pass them in `belief_ids` but expect non-IN or trivial antecedent counts to be dropped without notice.
- The function is emissive right now; if a caller wants to convert findings into mutations, that must be done explicitly (this module does not).

## 9. Dependencies

- `reasons/review_justifications.py:_process_batch` — does the per-batch LLM invocation and parsing.
- `reasons/review_justifications.py:_has_multi_antecedent_sl` — the filter predicate.
- `reasons/review_justifications.py:parse_justification_review` — tolerant JSON extraction from LLM output.
- `reasons.llm.invoke_model` — external model call.
- `reasons.review.format_belief_for_review` — prompt-format helper for a single belief.
- `REVIEW_JUSTIFICATIONS_PROMPT` / `REVIEW_BATCH_SIZE` constants.
- Standard library: `json`, `sys`, `concurrent.futures` (imported lazily inside `parallel > 0`).

## Topics to Explore

- [function] `reasons/review_justifications.py:_process_batch` — Shows how a batch of beliefs is formatted, sent to the LLM, and parsed; the unit of failure that `review_justifications` catches.
- [function] `reasons/review_justifications.py:_has_multi_antecedent_sl` — The gating predicate; understanding it clarifies exactly which beliefs reach the LLM.
- [function] `reasons/review.py:format_belief_for_review` — Defines the prompt shape; determines what the LLM actually sees per belief.
- [file] `reasons/llm.py` — Where `invoke_model` lives; behavior under timeout/retry errors feeds directly into this function's error semantics.
- [file] `tests/test_review_justifications.py` — Documents expected behavior (batching, parallel, callback, filtering) and any edge cases guarded by tests.

## Beliefs

- `review-justifications-is-read-only` — The function never mutates `nodes` or the underlying database; it only returns advisory results.
- `parallel-results-are-unordered` — When `parallel > 0`, returned results follow batch completion order, not the input `belief_ids` order.
- `belief-ids-is-weakly-honored` — Explicit `belief_ids` are still re-filtered by `truth_value == "IN"` and multi-antecedent SL, so non-matching IDs are silently dropped.
- `batch-errors-are-swallowed` — Any exception from `_process_batch` or `on_batch` is logged to stderr and never re-raised, so the function only propagates errors from argument construction (e.g. `batch_size <= 0`).
- `on-batch-receives-cumulative-list` — `on_batch` is called with the same growing `all_results` list each iteration; callers must copy if they intend to retain snapshots.
