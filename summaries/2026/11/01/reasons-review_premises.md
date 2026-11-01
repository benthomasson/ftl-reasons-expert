---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/review_premises.py

# `reasons/review_premises.py` — Premise Source-Verification Audit

## Purpose

This module closes a critical trust gap in the Truth Maintenance System. In a TMS, **premises are axioms** — they have no antecedent nodes to validate against, so the system treats them as ground truth. But those premises were originally extracted from source documents by an earlier LLM pass, and that extraction may have introduced fabricated details, overgeneralizations, or misreadings.

`review_premises` owns the responsibility of **fact-checking premises against their source material**. It sends each premise alongside its source document text to an LLM, which scores it on two axes: *accuracy* (does the source actually say this?) and *scoping* (is the claim stated with appropriate generality?). The output is a structured verdict per premise, which downstream consumers (like `repair_premises.py`) can use to fix or flag bad premises.

## Key Components

### Constants

| Name | Value | Role |
|------|-------|------|
| `REVIEW_BATCH_SIZE` | `5` | Default number of premises sent per LLM call. Keeps prompts short enough to stay within context limits while amortizing API costs. |

### `REVIEW_PREMISES_PROMPT` (template string)

A heavily-constrained system prompt that tells the LLM:
- You are auditing premises in a TMS.
- Evaluate **accuracy** (5 error types: `misread_source`, `overgeneralized`, `fabricated`, `unsupported`, or `null`) and **scoping**.
- Return **only** a JSON array with a strict schema.
- Be rigorous — a claim that *sounds* reasonable but isn't in the source is **not** accurate.
- Focus on the source document, not general knowledge.

The prompt uses `{{` / `}}` escaping for the JSON example so that `str.format(premises=...)` works correctly.

### `format_premise_for_review(node_id, nodes, source_content)`

**Contract:** Takes a single node ID, the full node dict, and the source text. Returns a markdown-formatted string block for that premise, or `""` if the node doesn't exist.

Output shape:
```
### premise-id
Claim: <text>
Source reference: <path>

Source document content:
```
<full source text>
```
```

This is the unit of text that gets concatenated into the prompt.

### `parse_premise_review_response(response)`

**Contract:** Takes raw LLM text, finds the first valid JSON array, and returns a list of normalized result dicts. Returns `[]` if nothing parseable is found.

Notable: it scans character-by-character for `[`, then uses `json.JSONDecoder.raw_decode()` to attempt parsing at each position. This is a resilience pattern — LLMs often wrap JSON in prose or code fences, so we can't just `json.loads(response)`. It also filters out items that lack an `"id"` key.

### `_process_batch(batch, nodes, source_contents, model, timeout)`

**Contract:** Takes one batch (list of premise IDs) and returns a list of result dicts. This is the worker function for both sequential and parallel execution paths.

Internally it:
1. Formats each premise via `format_premise_for_review`
2. Joins them into the prompt template
3. Calls `invoke_model(prompt, model, timeout)`
4. Parses and returns the results

### `review_premises(nodes, premise_ids, source_contents, model, timeout, batch_size, on_batch, parallel)`

**Contract (main entry point):**

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `nodes` | `dict[str, dict]` | — | Node data from `export_network()` |
| `premise_ids` | `list[str]` | — | IDs to review |
| `source_contents` | `dict[str, str]` | — | Maps source path → full text |
| `model` | `str` | `"claude"` | LLM model identifier |
| `timeout` | `int` | `300` | Seconds per LLM call |
| `batch_size` | `int` | `5` | Premises per LLM invocation |
| `on_batch` | `Callable[[list], None]` | `None` | Callback after each batch (for progress/streaming) |
| `parallel` | `int` | `0` | Concurrent workers (0 = sequential) |

**Returns:** `list[dict]` — flat list of all results across all batches.

## Patterns

1. **Batch-and-callback:** Work is split into fixed-size batches. After each batch completes, `on_batch(all_results)` fires with the *accumulating* list (not just the current batch). This lets callers render progress or do incremental repair without polling.

2. **Lazy parallelism:** `ThreadPoolExecutor` is imported *inside* the `if parallel > 0` branch. Sequential mode (the default) has zero concurrency overhead and no thread-safety concerns.

3. **Source-grouped ordering:** Before batching, premises are regrouped by their `source` field (the `by_source` dict). This means premises drawn from the *same* document tend to land in the *same* batch, so the LLM sees one source document repeated rather than interleaving 5 different documents in one prompt. This is a prompt-efficiency optimization.

4. **Defensive JSON parsing:** `parse_premise_review_response` doesn't assume clean output. It scans for `[`, attempts `raw_decode` at each position, and silently skips malformed items. This is the standard pattern for LLM output parsing in this codebase (compare with `review_justifications.py`).

5. **Mirror package:** `reasons_lib/review_premises.py` exists as a parallel copy (the repo has a `reasons/` and `reasons_lib/` split, likely for an internal vs. library distribution boundary). Changes here must be mirrored there.

## Dependencies

**Imports (this file →):**
- `json` — JSON decoding for LLM response parsing
- `sys` — `sys.stderr` for progress logging
- `.llm: invoke_model` — the single LLM invocation primitive shared across the project
- `concurrent.futures: ThreadPoolExecutor, as_completed` — lazy import for parallel mode

**Imported by (→ this file):**
- `reasons/api.py` — the public API layer that exposes `review_premises` to external callers
- `reasons_lib/review_premises.py` — the mirror copy (likely a `from ..reasons.review_premises import *` or re-export)
- `tests/test_review_premises.py` — direct unit tests
- `tests/test_repair_premises.py` — tests for the repair flow that *consumes* review output

## Flow

```
review_premises(nodes, premise_ids, source_contents, ...)
│
├─ 1. Group premise_ids by their source path  →  by_source: {path: [pids]}
│
├─ 2. Flatten into ordered_ids (preserves source grouping)
│
├─ 3. Slice into batches of `batch_size` (default 5)
│
├─ 4a. Sequential (parallel=0):
│      for each batch:
│        _process_batch(batch, ...)
│          ├─ format_premise_for_review() × N  →  premises_text
│          ├─ REVIEW_PREMISES_PROMPT.format(premises=...)
│          ├─ invoke_model(prompt, model, timeout)  →  raw text
│          └─ parse_premise_review_response(raw)  →  [result dicts]
│        all_results.extend(results)
│        on_batch(all_results) if set
│
├─ 4b. Parallel (parallel>0):
│      ThreadPoolExecutor(max_workers=parallel)
│      submit all batches → futures
│      as_completed(futures) → collect results, print progress, call on_batch
│
└─ 5. Return all_results (flat list)
```

The data transformation is: **structured node data + source text → LLM prompt → raw text → parsed JSON array → normalized result dicts**.

## Invariants

1. **One result per input premise.** The prompt instructs the LLM to return exactly one object per premise, in order. The parser enforces this by requiring an `"id"` field on each item.

2. **Mutual exclusion of `accurate` and `error_type`.** If `accurate` is `true`, `error_type` must be `null`. If `accurate` is `false`, `error_type` must be one of the four non-null values. The prompt enforces this; the parser does *not* re-validate it (it trusts the LLM).

3. **Source grouping precedes batching.** Premises from the same source are always contiguous in `ordered_ids`. Batches may span source boundaries at the edges, but within a batch the same source dominates.

4. **Idempotent and read-only on inputs.** `nodes` and `source_contents` are never mutated. The function is pure w.r.t. its inputs (aside from the LLM call and the `on_batch` callback).

5. **`source_contents` is keyed by the node's `source` field.** The lookup `source_contents.get(nodes[pid].get("source", ""), "(source not available)")` means if a premise has no source or the source isn't in the dict, the LLM sees the literal string `"(source not available)"` and should mark it `unsupported`.

## Error Handling

- **LLM call failure:** `invoke_model` may raise (timeout, network, rate limit). This propagates up to the `try/except` in the main loop, where it's caught, logged to `stderr` as `WARN: batch N failed: <e>`, and **execution continues** with the next batch. A single bad batch doesn't abort the entire review.

- **Unparseable LLM response:** `parse_premise_review_response` returns `[]` rather than raising. If the LLM returns prose instead of JSON, that batch simply contributes zero results. No exception, no retry.

- **Missing node:** `format_premise_for_review` returns `""` if `nodes.get(node_id)` is `None`. The empty string gets concatenated into the prompt (harmless), and the LLM simply has no claim text to evaluate for that slot.

- **Missing source content:** The `.get(..., "(source not available)")` fallback ensures the prompt is still well-formed. The LLM should flag the premise as `unsupported`.

- **Parallel mode exceptions:** `future.result()` re-raises whatever the worker raised. Caught by the same `try/except` in the `as_completed` loop.

- **No retry logic.** Failed batches are not retried. The caller (e.g., `api.py` or a CLI) is responsible for re-invoking if it needs completeness.

---

## Topics to Explore

- [file] `reasons/repair_premises.py` — The downstream consumer of this module's output; shows how `fabricated`/`overgeneralized` verdicts get turned into corrective updates to the TMS.
- [function] `reasons/llm.py:invoke_model` — The single LLM call primitive; understanding its timeout, retry, and model-routing behavior is essential before debugging batch failures here.
- [file] `tests/test_review_premises.py` — The test suite for this module; reveals the expected edge cases (missing sources, malformed LLM output, parallel execution) and the fixture structure.
- [file] `reasons/review_justifications.py` — The sibling module that audits justification nodes; comparing the two shows where the shared prompt/parsing pattern diverges for the justification-specific axes.
- [general] `source-content-resolution` — How `source_contents` gets populated (file reads, API fetches, etc.) is critical context; this module assumes the dict is already fully resolved before being called.

## Beliefs

- `review-premises-is-read-only` — `review_premises` never mutates `nodes` or `source_contents`; it is a pure audit step whose only side effect is the `on_batch` callback and stderr logging.
- `review-premises-batches-by-source` — Premises sharing the same `source` path are always placed in contiguous positions in the batch order, optimizing LLM context reuse within a single prompt.
- `review-premises-swallow-batch-failures` — A failed LLM call in any batch is logged to stderr and skipped; the function still returns a (possibly incomplete) results list and never raises.
- `review-premises-produces-repair-input` — The output schema (`id`, `accurate`, `well_scoped`, `error_type`, `comment`) is the exact input contract consumed by `repair_premises.py` to generate corrective TMS updates.
- `reasons-and-reasons-lib-are-mirrors` — `reasons/review_premises.py` and `reasons_lib/review_premises.py` must be kept in sync; changes to one without the other will cause behavioral divergence between the internal and library distributions.
