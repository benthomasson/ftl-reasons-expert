---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Understand where `review_results` come from and how `action == "error"` or `retract` should be handled by the caller (retraction semantics in the TMS).

## Where `review_results` come from

`review_results` are **LLM audit outputs**, not TMS retraction events. They are produced by three review modules and then consumed by repair/API code.

### 1. Belief review: `reasons/review.py:review_beliefs`

`review_beliefs(...)` selects derived IN beliefs with justifications, batches them, calls `invoke_model`, and parses the LLM JSON via `parse_review_response`.

Each result is normalized to a list of dicts shaped like:

```python
{
    "id": "...",
    "valid": bool,
    "sufficient": bool,
    "necessary": bool,
    "unnecessary_antecedents": [...],
    "comment": "...",
    "scope_findings": [...],
    "missing_property": "...",
    "defeat_reason_type": "...",
}
```

`parse_review_response` defaults `valid`, `sufficient`, and `necessary` to `True` when omitted. So a malformed or partial LLM response can look valid unless the caller checks IDs and completeness.

### 2. Premise review: `reasons/review_premises.py:review_premises`

`review_premises(...)` returns a list of dicts shaped like:

```python
{
    "id": "...",
    "accurate": bool,
    "well_scoped": bool,
    "error_type": ...,
    "comment": "...",
}
```

`reasons/api.py:4374-4384` shows premise review consumption using `data.get("results", [])` or `review_result.get("results", [])`, then filtering on `accurate`:

```python
inaccurate = [r for r in review_results_list if not r.get("accurate", True)]
```

### 3. Justification review: `reasons/review_justifications.py:review_justifications`

This returns dicts with:

```python
{
    "id": "...",
    "classification": "ALL" | "ANY" | "MIXED",
    "required_antecedents": [...],
    "independent_antecedents": [...],
    "comment": "...",
}
```

### Consumer shape mismatch

`reasons/repair.py:repair_beliefs` expects `review_results` as a **list** and filters invalid entries:

```python
invalid = [r for r in review_results if not r.get("valid", True)]
```

`reasons/repair_premises.py:repair_premises` expects `review_results` as a **dict keyed by node_id**:

```python
review_results: Dict of node_id -> review result dict.
```

and `_repair_one` does:

```python
review = review_results.get(node_id, {})
```

So callers must not assume all `review_results` have one shape. `reasons/api.py:4420` passes `review_results=review_map`, indicating the API layer builds a map for premise repair.

## How `action == "error"` or `retract` should be handled

Based only on these observations: **this cannot be answered directly, because the actual retraction API and `action == "error"` semantics are not present in the observed code.**

The observations show:

- `find_error_action_handling` found **no matches** for `action == "error"` handling.
- `find_retract_semantics` reported **no matches**.
- None of the review parsers emit an `action` key.
- Review failures are handled by swallowing batch exceptions and logging a warning, e.g. in `review_beliefs`:

```python
except Exception as e:
    print(f"  WARN: batch {batch_num} failed: {e}", file=sys.stderr)
```

So a failed review batch **omits results** rather than returning `{"action": "error"}`.

The only visible retraction semantics are in prompts, not implementation:

- `reasons/review_justifications.py` explains ALL vs ANY justifications:
  - ALL: if any antecedent is retracted, the conclusion is retracted.
  - ANY: the conclusion survives as long as at least one antecedent holds.
- `reasons/derive.py` explains TMS cascade behavior:
  - Retracting a base premise cascades derived conclusions OUT.
  - Outlist-gated conclusions are OUT while their outlist node is IN, and flip IN when that node goes OUT.

But the code that actually performs `retract` or interprets `action == "error"` is **missing from the observations**.

### Practical caller guidance from the observed code

- For belief review results, inspect `valid=False` entries and pass a **list** to `repair_beliefs`.
- For premise review results, inspect `accurate=False` entries and pass a **dict keyed by node_id** to `repair_premises`.
- For justification review results, use `classification` to decide whether an ALL justification should be treated as ANY/MIXED.
- If a review batch fails, expect **missing IDs**, not an error sentinel. Compare returned IDs to requested IDs if completeness matters.
- Do **not** invent an `action == "error"` branch unless you find the producer of that field. It is not produced by the observed parsers.

## What is missing

To fully answer the `action == "error"` / `retract` question, the following must be inspected:

- The TMS core retraction method, likely outside the observed files.
- The API/CLI command handler that produces or consumes `action`.
- `reasons/llm.py:invoke_model`, to see whether LLM errors can surface as structured action values.
- The database/network mutation layer that propagates OUT status through justifications and outlists.

## Topics to Explore

- [file] `reasons/api.py` — Shows API-level wiring for review result filtering and passing `review_map` into premise repair.
- [function] `reasons/repair.py:repair_beliefs` — Consumes list-shaped `review_results` and filters `valid=False`; clarifies repair-side expectations.
- [function] `reasons/repair_premises.py:repair_premises` — Consumes dict-shaped `review_results` keyed by `node_id`; highlights the caller shape mismatch.
- [general] `tms-retraction-core` — Find the actual retract/OUT propagation implementation; it is not included in the observations.
- [function] `reasons/llm.py:invoke_model` — Entry point for LLM failures that review batch handlers currently catch and log as warnings.

## Beliefs

- `review-beliefs-produces-validity-list` — `review_beliefs` returns a list of normalized dicts containing `valid`, `sufficient`, `necessary`, `scope_findings`, `missing_property`, and `defeat_reason_type`.
- `review-premises-produces-accuracy-list` — `review_premises` returns a list of dicts containing `accurate`, `well_scoped`, and `error_type`.
- `repair-premises-requires-review-results-map` — `repair_premises` expects `review_results` as a dict keyed by node ID, while `repair_beliefs` expects a list.
- `review-batch-errors-omit-results` — Batch exceptions in review modules are logged as warnings and skipped, producing partial or empty results rather than an `action == "error"` item.
- `review-parsers-do-not-emit-action-field` — The observed review parsers do not produce an `action` key.
