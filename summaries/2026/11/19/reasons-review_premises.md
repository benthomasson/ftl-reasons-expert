---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/review_premises.py

# `reasons/review_premises.py` — Premise Factual-Accuracy Reviewer

## Purpose

This module is the "source-truthing" pass over the TMS (Truth Maintenance System) graph. Premises are **axioms** — the system trusts them without derivation, so they can silently absorb hallucinations, overgeneralizations, or misreadings introduced by whichever extraction LLM originally wrote them into the network.

`review_premises()` closes that loop: it round-trips each premise back to the raw text of the source document it claims to come from, asks an LLM "does this source actually support this claim?", and returns structured per-premise verdicts. Those verdicts are presumably consumed by `reasons/repair_premises.py` to flag, rewrite, or reject bad premises.

The module is deliberately decoupled from storage: it takes plain dicts from `export_network()` and a source-content map, and returns plain dicts. It has no dependency on `storage.py`, `network.py`, or `pg.py`.

## Key Components

### `REVIEW_BATCH_SIZE = 5`
Default number of premises sent in a single LLM call. Keeping it small is a reliability choice — the JSON-array response format is easy for a model to corrupt when the array gets long, and a failed batch is entirely swallowed (see Error Handling).

### `REVIEW_PREMISES_PROMPT`
A `str.format`-string (note the doubled braces `{{` / `}}` around the JSON example) with one substitution point: `{premises}`. It is the contract between this module and the LLM:

- **Two axes** per premise: `accurate` (bool) and `well_scoped` (bool).
- **Closed error taxonomy**: `error_type ∈ {"misread_source", "overgeneralized", "fabricated", "unsupported", None}`.
- **Invariant the LLM must obey**: `accurate=false ⇒ error_type != null`, and `accurate=true ⇒ error_type == null`. This is specified in the prompt but *not re-validated in Python*.
- **Ordering contract**: "same order as presented" — the parser honors this by trusting `id` matching rather than positional matching (see below).

### `format_premise_for_review(node_id, nodes, source_content) -> str`
Renders one premise as a markdown subsection:

```
### {id}
Claim: <node.text>
Source reference: <node.source>
Source document content:
```
<source_content>
```
```

- Returns `""` (empty string) if `node_id` is missing from `nodes`. That empty string is later joined into the batch prompt, silently producing a premise with no content.
- `Source reference` is emitted only when `node["source"]` is non-empty.

### `parse_premise_review_response(response) -> list[dict]`
Tolerant JSON extractor. Because LLMs often wrap their array in prose or ``` fences, this walks every index in the raw response, tries `json.JSONDecoder().raw_decode(response, i)`, and returns the first successful **list** whose items are dicts with `"id"`.

- Entries lacking `"id"` are dropped.
- Unknown fields are dropped (only the 5 documented keys survive).
- Defaults: `accurate=True`, `well_scoped=True`, `comment=""`, `error_type=None`. So a malformed object silently becomes an "accurate" premise — the safest default given `on_batch` may persist these.
- Returns `[]` if nothing parses.

### `_process_batch(batch, nodes, source_contents, model, timeout) -> list[dict]`
Format → `invoke_model` → `parse_premise_review_response`. The source content is resolved via `source_contents.get(nodes[pid].get("source",""), "(source not available)")`, so a premise pointing at an unfetched source still gets a distinctive placeholder string (the model will presumably flag that as unsupported).

**Docstring/contract mismatch:** the docstring says "Returns `(batch_results, error_or_None)`" but the code returns only the list. Callers match the implementation, not the docstring — worth fixing.

### `review_premises(nodes, premise_ids, source_contents, model="claude", timeout=300, batch_size=5, on_batch=None, parallel=0) -> list[dict]`
Main entry point.

| Argument | Contract |
|---|---|
| `nodes` | `{node_id: node_dict}` from `export_network()`; each node should have `text` and `source`. |
| `premise_ids` | Premise node IDs to audit. **Not** guaranteed to be present in `nodes`. |
| `source_contents` | `{source_path: raw_text}`. Missing entries fall back to `"(source not available)"`. |
| `model` / `timeout` | Passed straight to `invoke_model`. |
| `batch_size` | Chunk size; must be ≥ 1 (no validation). |
| `on_batch` | Called with the **cumulative** `all_results` list after each batch. Note: same list object, mutated in place — a callback that retains it will see later mutations. |
| `parallel` | `0` → sequential. `>0` → `ThreadPoolExecutor(max_workers=parallel)`. |
| returns | Flat list of review-result dicts. |

## Patterns & Idioms

- **Group-by-source before batching.** After early-return, all premises are bucketed into `by_source[source_path] = [pids...]`. This keeps same-source premises in the same batch, so (a) source text is effectively deduplicated within prompt construction, and (b) comparisons against the same source are consistent. Because the bucket dict preserves insertion order in Python 3.7+, the *initial order of first occurrence* of each source governs the global ordering.
- **Flatten-and-chunk.** `ordered_ids` is the concatenation of bucket values, then sliced `range(0, n, batch_size)`. This intentionally *reorders* premises; see Invariants.
- **Tolerant LLM-JSON parsing.** `raw_decode` scanning is preferred over `json.loads(response)` so the model can "think out loud" before emitting JSON.
- **Callback-based progress.** `on_batch(all_results)` provides incremental reporting without forcing this module to know about persistence.
- **Fail-soft per batch.** A single failed batch (network error, bad JSON, LLM refusing) is logged to stderr and does not abort the whole run.

## Dependencies

- `json`, `sys` — standard library.
- `from .llm import invoke_model` — the only intra-repo runtime import.
- `from concurrent.futures import ThreadPoolExecutor, as_completed` — **imported lazily inside the `parallel > 0` branch**, so importing this module for sequential use costs no thread-pool machinery.
- **Imported by**: `reasons/api.py`, `reasons_lib/review_premises.py`, `tests/test_repair_premises.py`, `tests/test_review_premises.py`. Notice `reasons_lib/review_premises.py` is not in the import list of this file — likely the vendored/installed twin has an identical import graph.

## Flow

1. `premise_ids` empty → return `[]` immediately.
2. Build `source_for_premise[pid] = nodes[pid]["source"]` for every known node. Unknown IDs are simply absent from this dict.
3. Bucket by source via `by_source.setdefault(src, []).append(pid)`. Unknown premise IDs get bucket `""` (empty string).
4. Flatten buckets into `ordered_ids` — **order is by first-occurrence source, not input order**.
5. Slice into `batches`.
6. **Sequential path (`parallel == 0`)**:
   - Log `Reviewing batch X/N (k premises)...`
   - `try: results = _process_batch(...)` → extend → `on_batch(all_results)`.
   - `except: print WARN, continue`.
7. **Parallel path (`parallel > 0`)**:
   - Submit a future per batch; `as_completed` yields them in **completion order**, so final `all_results` is non-deterministic.
   - Same try/except per future; `on_batch` fired on every successful completion (in completion order).
8. Return `all_results`.

## Invariants & Assumptions

- **Results are keyed by `id`, not by position.** The prompt asks for same-order, but the parser relies on `id` because prompt order is already violated by grouping.
- **The result list may be out of order relative to `premise_ids`.** Callers should join on `id`, not zip.
- **Unknown premise IDs are dropped silently** — they produce an empty formatted chunk in the batch text but produce no matching LLM result, so they vanish from the output. There is no warning.
- **`error_type` set ⇔ `accurate=false`** is enforced only by the prompt; the Python layer does not validate this and will happily pass through `{"accurate": false, "error_type": null}` if the model returns it.
- **`on_batch` is called with the shared mutable `all_results` list.** Do not assume a snapshot.
- **`batch_size` is assumed ≥ 1.** `range(0, n, 0)` would raise `ValueError`; there is no guard.
- **`parallel` threads perform network I/O concurrently** — nothing serializes stderr writes, so interleaved `WARN` lines are possible.

## Error Handling

| Failure | Behavior |
|---|---|
| Missing node for a premise ID | `format_premise_for_review` returns `""`; premise still occupies a slot in the batch; result silently missing. |
| Source not in `source_contents` | Rendered as `"(source not available)"`; model decides. |
| LLM response is prose-only or truncated | `parse_premise_review_response` returns `[]` for the batch. Batch counted as "complete" with zero results — **no warning**. |
| `invoke_model` raises (timeout, rate limit, network) | Caught in `review_premises`; `WARN: batch N failed: ...` to stderr; batch skipped. |
| Concurrent batch raises | Same swallow, via `future.result()`. |
| Callback raises | **Not caught** inside the sequential/parallel loop — a raising `on_batch` will abort the whole function (and, in the parallel branch, propagate out of the `with` block, cancelling pending futures). |

The design philosophy is explicit: **partial results beat a failed audit**. The trade-off is that a badly-degraded run (e.g., LLM returning empty strings) is indistinguishable, at this layer, from a clean run over fewer premises — downstream verification is required.

---

## Topics to Explore

- [function] `reasons/llm.py:invoke_model` — Inspect the timeout/retry behavior and return type contract this module relies on.
- [file] `reasons/repair_premises.py` — See how `review_premises` output drives actual repairs; this is the downstream consumer of `error_type`.
- [file] `reasons/api.py` — Understand whether the API layer enforces any validation on `premise_ids` (e.g., that they really are premises) before delegating here.
- [function] `reasons/review_justifications.py:review_justifications` — Sibling reviewer with an analogous batching/callback structure; useful to compare conventions.
- [file] `tests/test_review_premises.py` — Confirms the intended ordering, callback semantics, and whether out-of-order/unknown-ID cases are covered.

## Beliefs

- `review-premises-groups-by-source` — Premises are bucketed by `node["source"]` and flattened before batching, so final result order follows first-occurrence source, not `premise_ids` order.
- `parse-premise-review-response-scans-every-bracket` — `parse_premise_review_response` iterates every `[` in the response and uses `JSONDecoder.raw_decode`, returning the first parseable list with id-bearing dicts.
- `review-premises-swallows-batch-failures` — Exceptions from `_process_batch` (and from `future.result()` in parallel mode) are caught in `review_premises` and logged as `WARN`, never re-raised.
- `on-batch-receives-mutable-cumulative-list` — `on_batch` is invoked with the same `all_results` list object, mutated in place, after each batch.
- `process-batch-returns-list-not-tuple` — Despite its docstring saying `(batch_results, error_or_None)`, `_process_batch` returns a single `list[dict]` from `parse_premise_review_response`.
- `missing-premise-nodes-silently-omitted` — A `premise_id` absent from `nodes` produces an empty formatted chunk and no corresponding result entry, with no warning emitted.
