---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/repair_premises.py

# `reasons/repair_premises.py`

## Purpose

This module is the **repair stage of a premise-review pipeline** in a Truth Maintenance System (TMS). It consumes premises that a prior review step (`review-premises`) flagged as inaccurate, and dispatches each one to an LLM with the original claim, the review comment, and the raw source document. The LLM is asked to return one of two actions per premise:

- **`rewrite`** — produce a corrected claim that still matches the source
- **`retract`** — declare the claim unsupported by the source

It does *not* apply changes to the store itself — it only produces a list of repair verdicts that higher-level code (`reasons/api.py`) can then execute. So its responsibility is purely **the review→corrective-decision transform**.

It is mirrored at `reasons_lib/repair_premises.py` (likely the installable-package twin of the `reasons/` tree).

## Key Components

### `REPAIR_PREMISES_PROMPT` (constant)
A `str` template with a single `{premises}` placeholder (formatted via `.format(...)`). The literal prompt embeds a JSON schema example and explicit rules:

- one object per premise, same order as input
- `action ∈ {"rewrite", "retract"}`
- `corrected_text` is a complete replacement when rewriting, `null`/omitted when retracting
- `rationale` is one sentence
- **"Only use information from the source document. Do not add external knowledge."**

### `format_premise_for_repair(node_id, nodes, source_content, review_result) -> str`
Renders one premise block for the prompt. Contract:

- Returns `""` if `node_id` is not a key in `nodes`.
- Emits a `### {node_id}` heading, then `Original claim:`, `Error type:`, optional `Review comment:`, optional `Source reference:`, then the raw source wrapped in a triple-backtick fence.
- Pulls `error_type` (default `"inaccurate"`) and `comment` (default `""`) from `review_result`.
- Pulls `text` and `source` from `nodes[node_id]`.

### `parse_repair_response(response) -> list[dict]`
Robust extraction of the first valid JSON array from an LLM string. Algorithm:

1. Walk every character; on each `"["`, attempt `json.JSONDecoder().raw_decode(response, i)`.
2. Skip positions that raise `JSONDecodeError` or decode to a non-list.
3. For each element, require `isinstance(item, dict) and "id" in item`, else drop it.
4. Normalize each surviving item to `{"id", "action", "corrected_text", "rationale"}` with defaults `action="retract"`, `corrected_text=None`, `rationale=""`.
5. Return the first non-empty normalized list; fall back to `[]`.

This deliberately tolerates the common LLM failure mode of a leading prose sentence before the JSON, and silently drops malformed items rather than raising.

### `_repair_one(node_id, nodes, source_contents, review_results, model, timeout) -> dict`
Single-premise worker. Steps:

1. `review = review_results.get(node_id, {})`
2. `source = nodes[node_id].get("source", "")` — **note this index access is *outside* the try/except**
3. `source_text = source_contents.get(source, "(source not available)")`
4. Build the prompt via `format_premise_for_repair` + `REPAIR_PREMISES_PROMPT.format`.
5. `invoke_model(prompt, model=model, timeout=timeout)`, then `parse_repair_response`.
6. Return `parsed[0]` if present, else a synthetic **retract** with rationale `"Could not parse repair response"`.
7. On any exception from `invoke_model` or parsing, return `{"id", "action": "error", "corrected_text": None, "rationale": str(e)}`.

So the returned dict always has keys `id`, `action`, `corrected_text`, `rationale`, but `action` ranges over `{"rewrite", "retract", "error"}`.

### `repair_premises(nodes, premise_ids, source_contents, review_results, model="claude", timeout=300, parallel=0, on_result=None) -> list[dict]`
Public entry point.

- **Fast path**: empty `premise_ids` → `[]` (no LLM calls).
- **Parallel mode** (`parallel > 0`): `ThreadPoolExecutor(max_workers=parallel)`; submits `_repair_one` for every `pid`, dict-maps `future → pid`, iterates `as_completed`, logs `"  Repaired {i}/{n}: {pid}"` to stderr, appends result, calls `on_result(all_results)`.
- **Sequential mode** (`parallel == 0`): simple `for` loop with `"  Repairing {i}/{n}: {pid}..."` logs and the same append/callback pattern.
- Returns the accumulated `all_results` list.

`on_result` receives the **whole growing list**, not just the new result, so UIs can re-render partial progress.

## Patterns

- **Prompt-template + structured JSON contract** — the LLM is a pure function of `(claim, review, source) → verdict`, mediated by a strict schema.
- **Robust JSON-in-text parsing** (`raw_decode` scanning) — a standard idiom for making LLM output parseable when the model "helpfully" wraps the JSON in prose.
- **Strategy-by-field** — a single result type with a discriminator (`action`) rather than separate Rewrite/Retract classes.
- **Fan-out / fan-in with thread pool** — `ThreadPoolExecutor` + `as_completed` for parallel LLM calls; the callback (`on_result`) is a small Observer pattern for streaming progress.
- **Error-as-data** — regardless of what goes wrong in `_repair_one`, the result is a dict rather than a raised exception.
- **Per-item defaulting** — `dict.get(..., default)` is used uniformly for `review_results`, `nodes[…]`, `source_contents`, and every JSON field.

## Dependencies

**Imports (inbound):**
- `json`, `sys`, `concurrent.futures.{ThreadPoolExecutor, as_completed}` (stdlib)
- `from .llm import invoke_model` — the only domain dependency; the module is agnostic to which provider is behind `model`.

**Imported by (outbound):**
- `reasons/api.py` — exposes `repair_premises` on the public API surface.
- `reasons_lib/repair_premises.py` — the mirrored package copy.
- `tests/test_repair_premises.py` — the unit-test target.

The module knows nothing about storage, network, or the review step that produced `review_results`; its inputs come pre-materialized (nodes dict, already-loaded source text, review dict), which makes it easy to test in isolation.

## Flow

```
caller (api.py)
   │
   ▼
repair_premises(nodes, premise_ids, source_contents, review_results, ...)
   │
   ├─ premise_ids empty? → return []
   │
   ├─ parallel > 0? ──► ThreadPoolExecutor
   │                       submit _repair_one(pid, ...) for each pid
   │                       as_completed:
   │                          result = future.result()
   │                          all_results.append(result)
   │                          on_result(all_results)
   │
   └─ parallel == 0 ► for pid in premise_ids:
                          result = _repair_one(...)
                          all_results.append(result)
                          on_result(all_results)

_repair_one(pid, ...)
   │
   ├─ review = review_results.get(pid, {})
   ├─ source = nodes[pid]["source"]        ◄── KeyError possible
   ├─ source_text = source_contents.get(source, "(source not available)")
   ├─ prompt = format_premise_for_repair(...) + REPAIR_PREMISES_PROMPT.format
   ├─ try:
   │     response = invoke_model(prompt, model, timeout)
   │     parsed = parse_repair_response(response)
   │     return parsed[0] or {retract…}
   └─ except Exception e:
         return {"action": "error", "rationale": str(e), ...}
```

## Invariants

- **Empty input short-circuit**: `premise_ids == []` ⇒ return value is `[]`, with zero side effects and zero LLM calls.
- **Result shape is uniform**: every element has exactly `id`, `action`, `corrected_text`, `rationale`.
- **Action domain**: at the `_repair_one`/public boundary, `action ∈ {"rewrite", "retract", "error"}` (the prompt-level domain is only rewrite/retract; `"error"` is injected by the worker).
- **Parse-drop rule**: `parse_repair_response` never raises — malformed items are dropped, and an unparseable response yields a fallback retract.
- **`on_result` is called after every result** (both modes), and always receives the full accumulated list.
- **Parallel order is non-deterministic**: `as_completed` yields futures in completion order, so `all_results` and the progress log are not in `premise_ids` order. Callers wanting deterministic order must sort by `id` or use sequential mode.
- **Prompt template must have exactly one `{premises}` placeholder** — otherwise `.format(premises=…)` raises `KeyError`/`IndexError`.

## Error Handling

Three tiers, all of them *fail-soft* at the result boundary but with one gap:

1. **LLM/parse errors are swallowed as data.** Everything inside the `try` in `_repair_one` is caught by `except Exception as e`, producing `{"action": "error", "rationale": str(e)}`. Callers can inspect `action == "error"` but the rest of the batch proceeds.
2. **Parse failures are silently downgraded to retractions.** If `invoke_model` returns something `parse_repair_response` can't decode, the premise is returned as `action: "retract"` with rationale `"Could not parse repair response"`. This is a policy decision — a malformed LLM response results in **deletion**, so a caller should probably guard against a high rate of these.
3. **Pre-try access can still raise.** `nodes[node_id].get("source", "")` is outside the `try`, so a `premise_ids` value not present in `nodes` raises `KeyError`. In parallel mode this surfaces via `future.result()` *inside* the `as_completed` loop and aborts the whole batch (already-completed results are lost because `all_results` is local). `format_premise_for_repair`'s `if not node: return ""` guard cannot save this path because `_repair_one` touches `nodes[node_id]` first. Callers are expected to pass `premise_ids ⊆ nodes`.

Additionally, progress lines are written to `sys.stderr` (not stdout) so they don't pollute machine-readable output of the calling CLI.

## Topics to Explore

- [file] `reasons/llm.py` — `invoke_model` is the only external contract this module depends on; understand retries, provider selection, and how `model`/`timeout` are interpreted.
- [file] `reasons/review_premises.py` — Produces the `review_results` dict consumed here; confirming its `error_type`/`comment` schema is essential for maintaining `format_premise_for_repair`.
- [file] `reasons/api.py` — Shows how `repair_premises` is wired to storage: whether results are applied as edit/retract operations, and whether `on_result` is used for streaming to a UI or CLI.
- [function] `reasons/repair_premises.py:parse_repair_response` — The `raw_decode` scan is the most subtle piece; worth reading alongside `tests/test_repair_premises.py` to see which malformed LLM outputs are covered.
- [general] `reasons-vs-reasons_lib-mirroring` — Both trees contain a copy of this file; verify the sync mechanism (copy script, symlink, or generated package) before editing.

## Beliefs

- `empty-premise-ids-returns-empty-list` — `repair_premises` returns `[]` immediately when `premise_ids` is empty, with no LLM calls.
- `llm-failures-are-swallowed-as-error-actions` — Exceptions inside `_repair_one`'s `try` block are converted to `{"action": "error", "rationale": str(e)}` rather than re-raised.
- `parse-failures-downgrade-to-retract` — An unparseable LLM response yields `{"action": "retract", "rationale": "Could not parse repair response"}`.
- `parallel-mode-preserves-completion-order-not-input-order` — Parallel results are appended in `as_completed` order, so `all_results` may not match the order of `premise_ids`.
- `repair-one-has-unguarded-node-lookup` — `nodes[node_id]` is accessed outside the `try`, so a missing node id raises `KeyError` and, in parallel mode, aborts the `as_completed` loop.
- `on-result-receives-accumulating-list` — The `on_result` callback is invoked with the full growing `all_results` list after each item, not with a single new result.
- `parse-repair-response-scans-for-first-valid-json-array` — Parsing iterates over each `[` position and returns the first list containing at least one dict with an `id` key, dropping malformed entries.
