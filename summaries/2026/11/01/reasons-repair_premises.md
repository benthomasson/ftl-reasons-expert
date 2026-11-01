---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/repair_premises.py

# `reasons/repair_premises.py`

## Purpose

This module is the **repair execution layer** in a Truth Maintenance System (TMS). It sits in the pipeline *after* `review_premises` has flagged premises as inaccurate. Its sole job: for each flagged premise, either **rewrite** it to faithfully reflect the source document, or **retract** it entirely if no defensible version exists in the source.

It does *not* perform the review itself (that's `reasons/review_premises.py`), and it does *not* mutate the network graph (that's `reasons/repair.py` or `reasons/api.py`). It's a pure "ask the LLM to fix or kill this claim" step that returns structured results.

## Key Components

### `REPAIR_PREMISES_PROMPT` (constant)
The system prompt template. It:
- Instructs the model to return **only** a JSON array.
- Defines two allowed actions: `"rewrite"` and `"retract"`.
- Constrains the model to use *only* the provided source text (no external knowledge).
- Accepts a `{premises}` placeholder filled by `format_premise_for_repair`.

### `format_premise_for_repair(node_id, nodes, source_content, review_result)`
Builds the human-readable block for one premise that gets stuffed into the prompt. It assembles:
- Node ID, original claim text
- Error type + review comment (from the prior review step)
- Source reference path
- The full source document content in a fenced code block

Returns `""` if the node is missing from `nodes` — a silent no-op for that premise.

### `parse_repair_response(response)`
A **defensive JSON extractor**. It scans the raw LLM string character-by-character looking for the first `[`, then calls `json.JSONDecoder().raw_decode()` from that offset. This handles cases where the model wraps the JSON in prose or code fences.

Validation per item: must be a `dict` with an `"id"` key. Defaults `action` to `"retract"` if absent, `rationale` to `""`.

Returns `[]` if no valid JSON array is found.

### `_repair_one(node_id, nodes, source_contents, review_results, model, timeout)`
The per-premise worker. Steps:
1. Looks up the node, its source path, and the review result.
2. Resolves source text from `source_contents` (falls back to `"(source not available)"`).
3. Builds the prompt, calls `invoke_model`.
4. Parses the response; takes `parsed[0]` (expects exactly one result for one premise).
5. On parse failure: returns a `{"action": "retract", ...}` with rationale "Could not parse repair response".
6. On exception: returns `{"action": "error", ...}`.

### `repair_premises(nodes, premise_ids, source_contents, review_results, model, timeout, parallel, on_result)`
**Public entry point.** Orchestrates repair across all flagged premises.

- `parallel=0` → sequential loop with stderr progress.
- `parallel > 0` → `ThreadPoolExecutor(max_workers=parallel)` with `as_completed`.
- Accepts an `on_result(all_results)` callback invoked after *each* individual result lands (useful for live UI updates or incremental persistence).
- Returns the full list of result dicts.

## Patterns

| Pattern | Where |
|---|---|
| **Prompt-as-template** | `REPAIR_PREMISES_PROMPT` + `.format(premises=...)` — single source of truth for LLM instructions |
| **Defensive JSON parsing** | `parse_repair_response` scans for `[` and uses `raw_decode` — tolerates markdown fences, preamble text |
| **Graceful degradation** | Unparseable response → default to `"retract"` rather than raising. The TMS treats "couldn't verify" as "can't trust" |
| **Worker + orchestrator** | `_repair_one` is a pure per-item worker; `repair_premises` handles scheduling (sequential or thread-pooled) |
| **Callback for streaming** | `on_result` enables incremental consumption without waiting for the full batch |
| **Mirror package** | Identical copy lives in `reasons_lib/repair_premises.py` (likely a vendored/library distribution of the `reasons/` package) |

## Dependencies

**Imports (this file → others):**
- `json` — parsing
- `sys` — stderr progress output
- `concurrent.futures.ThreadPoolExecutor, as_completed` — parallel execution
- `.llm.invoke_model` — the LLM call boundary (wraps whichever provider is configured)

**Imported by (others → this file):**
- `reasons/api.py` — the public API surface that exposes repair as an endpoint
- `reasons_lib/repair_premises.py` — the library mirror (re-exports or duplicates)
- `tests/test_repair_premises.py` — unit/integration tests

## Flow

```
repair_premises(nodes, premise_ids, source_contents, review_results, ...)
│
├─ For each pid in premise_ids (sequential or threaded):
│   │
│   ├─ _repair_one(pid, ...)
│   │   │
│   │   ├─ Resolve source text:
│   │   │   nodes[pid]["source"] → source_contents[path] → text
│   │   │   (or "(source not available)")
│   │   │
│   │   ├─ format_premise_for_repair(...)
│   │   │   → "### pid\nOriginal claim: ...\nSource document content:\n```...```"
│   │   │
│   │   ├─ REPAIR_PREMISES_PROMPT.format(premises=above)
│   │   │
│   │   ├─ invoke_model(prompt, model, timeout)   ← LLM call
│   │   │
│   │   ├─ parse_repair_response(raw_string)
│   │   │   → [{"id": ..., "action": "rewrite"|"retract", "corrected_text": ..., "rationale": ...}]
│   │   │
│   │   └─ Return parsed[0]  (or fallback dict)
│   │
│   ├─ Append to all_results
│   ├─ Call on_result(all_results) if provided
│   └─ Print progress to stderr
│
└─ Return all_results
```

The caller (typically `reasons/api.py`) then takes those results and **applies** them to the TMS graph — replacing premise text on `"rewrite"` or removing the node on `"retract"`.

## Invariants

1. **One result per input premise.** The prompt explicitly demands "one object per premise, in the same order." `_repair_one` always returns exactly one dict (either parsed or fallback).
2. **Action is closed-set.** Valid values are `"rewrite"`, `"retract"`, or `"error"` (the latter only from exception paths). The LLM is instructed to only produce the first two.
3. **Source-only reasoning.** The prompt forbids external knowledge. A premise can only be rewritten to something the source text actually says.
4. **`corrected_text` is a complete replacement**, not a diff or patch. The TMS can swap the node's `text` field wholesale.
5. **Idempotency of parsing.** If the LLM returns malformed JSON, the premise is *retracted*, not left in limbo. This is a deliberate safety choice: a TMS would rather lose a premise than keep an unverifiable one.
6. **Order in prompt = order in list.** The prompt says "same order as presented," but with `parallel > 0` the *results list* may be in completion order, not input order. The `id` field is the stable join key.

## Error Handling

| Failure mode | Handling |
|---|---|
| Node not in `nodes` dict | `format_premise_for_repair` returns `""`; the premise still gets an LLM call with an empty prompt section (likely a wasted call, but no crash) |
| Source file not in `source_contents` | Substituted with `"(source not available)"` — LLM will almost certainly retract |
| LLM returns prose around JSON | `parse_repair_response` scans for `[` and uses `raw_decode` — tolerant of markdown fences |
| LLM returns invalid JSON entirely | `parse_repair_response` returns `[]` → `_repair_one` returns a `"retract"` fallback |
| LLM returns list with no `"id"` fields | Items are skipped; if all are skipped, falls through to the same `"retract"` fallback |
| `invoke_model` raises (timeout, network, etc.) | Caught by `except Exception` in `_repair_one`; returns `{"action": "error", ...}` — **not** re-raised to the caller |
| `future.result()` in parallel path | **Not** wrapped in try/except. If `_repair_one` somehow raised (it shouldn't, given the internal catch), the exception would propagate out of `repair_premises` and crash the batch. |

The philosophy: **never crash the batch because of one bad premise.** Every individual failure degrades to a safe default (`"retract"` or `"error"`), and the caller can inspect the `action` field to decide follow-up.

## Topics to Explore

- [file] `reasons/review_premises.py` — The upstream step that *produces* the `review_results` dict this module consumes; understanding its output schema (`error_type`, `comment`) clarifies what the repair prompt actually receives
- [file] `reasons/repair.py` — The sibling module that handles broader network repair (dangling dependents, nogood resolution); useful to see how `repair_premises` results are *applied* to the graph
- [function] `reasons/llm.py:invoke_model` — The LLM abstraction layer; knowing its contract (timeout semantics, retry behavior, model resolution) is critical for debugging flaky repair runs
- [file] `reasons/api.py` — The public API that wires `review_premises` → `repair_premises` → graph mutation; shows the full pipeline and how `on_result` is used for streaming
- [file] `tests/test_repair_premises.py` — Reveals the expected edge cases (empty premise list, missing sources, malformed LLM output, parallel execution) and the exact shape of result dicts

## Beliefs

- `repair-premises-degrades-to-retract-on-parse-failure` — If the LLM response cannot be parsed as a valid JSON array, `_repair_one` returns `{"action": "retract"}` rather than raising, ensuring the TMS never retains an unverifiable premise
- `repair-premises-consumes-review-premises-output` — The `review_results` parameter must be a `Dict[str, dict]` with keys `error_type` and `comment` per node ID, matching the output schema of `reasons/review_premises.py`
- `repair-premises-parallel-ordering-uses-id-not-index` — When `parallel > 0`, the returned list is in completion order, not input order; callers must join on the `"id"` field rather than positional index
- `repair-premises-swalls-llm-exceptions` — `_repair_one` catches all exceptions from `invoke_model` and converts them to `{"action": "error"}` dicts; the caller never sees a raised exception from the LLM call itself
- `repair-premises-is-source-only` — The prompt explicitly forbids the LLM from using external knowledge; a premise can only be rewritten to text supported by the provided `source_contents`, making this step a grounded repair rather than a generative one
