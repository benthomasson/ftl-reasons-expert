---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The four-way `_classify_review_result` precedence and how it affects downstream filtering (e.g., `list_nodes(never_reviewed=True)` ignores `review_result` entirely).

## `_classify_review_result` and downstream filtering

### What is actually observable

- `_classify_review_result(result: dict) -> str` is defined at `reasons/api.py:4049`.
- Its only observed call site is `reasons/api.py:4160`:
  ```python
  net.nodes[nid].metadata["review_result"] = _classify_review_result(result_map[nid])
  ```
  So the classifier collapses a review-result dict into one string and stores it under `metadata["review_result"]`.
- The input dict shape comes from `reasons/review.py:parse_review_response`, which emits:
  - `valid`
  - `sufficient`
  - `necessary`
  - `unnecessary_antecedents`
  - `comment`
  - `scope_findings`
  - `missing_property`
  - `defeat_reason_type`

### The exact four-way precedence is not visible

The observations do **not** include the body of `_classify_review_result`, only its definition line. Because of that, we cannot verify:

- which fields it branches on,
- the order of the branches,
- the exact returned strings,
- which condition wins when multiple booleans are false.

So the precise four-way precedence cannot be stated from the provided evidence. A likely shape would be something like “invalid wins over insufficient, which wins over unnecessary, otherwise sound,” but that is inference, not confirmed code. To answer definitively, we need `reasons/api.py` around lines `4049-4160`.

### How it affects downstream filtering

`review_result` is stored in node metadata, not as a dedicated column. In `reasons/storage.py`, the `nodes` table has `metadata_json` and `reviewed_at`, but no `review_result` column.

`list_nodes` is defined at `reasons/api.py:3392` and accepts `never_reviewed: bool = False` at `reasons/api.py:3402`. The observed `never_reviewed` handling is separate:

- `reasons/api.py:3429-3430`: `if never_reviewed: unsupported.append("never_reviewed")`
- `reasons/api.py:3473`: another `if never_reviewed:` block applies the filter

There is no observed reference to `metadata["review_result"]` inside `list_nodes`. That means `list_nodes(never_reviewed=True)` is a “has this node been reviewed at all?” filter, not a “does this node have a specific review classification?” filter. A node with `review_result="invalid"` is still reviewed, so it should not be returned by `never_reviewed=True`; a never-reviewed node has no meaningful `review_result` to filter on.

The CLI mirrors this separation: `reasons/cli.py:2069` passes `args.never_reviewed`, `reasons/cli.py:2079` treats `never_reviewed` and `not_reviewed_since` as mutually exclusive, and `reasons/cli.py:2092` uses them to decide whether to show review information. The tests at `tests/test_review.py:710-733` verify never-reviewed semantics, not classification semantics.

### Practical takeaway

`review_result` is an audit classification stored in metadata. `never_reviewed` is an orthogonal review-status filter. If you want to filter by `review_result`, you need a different query/filter that reads `metadata["review_result"]` or adds a dedicated column. That path is not present in the observed code.

## Topics to Explore

- [function] `reasons/api.py:_classify_review_result` — Read the full body to confirm the exact four-way branch order and returned labels.
- [function] `reasons/api.py:list_nodes` — Inspect lines 3392-3480 to see how `never_reviewed` interacts with the `unsupported` list and whether any review_result filter exists elsewhere.
- [file] `reasons/api.py` — Examine the review ingestion flow around line 4160 and how `result_map` is built.
- [file] `reasons/storage.py` — Confirm that `review_result` lives in `metadata_json` while `reviewed_at` is a dedicated column.
- [file] `tests/test_review.py` — Review the never-reviewed and not-reviewed-since tests for expected filtering behavior.

## Beliefs

- `review-result-stored-in-node-metadata` — `_classify_review_result` output is assigned to `net.nodes[nid].metadata["review_result"]` at `reasons/api.py:4160`.
- `review-result-is-not-dedicated-column` — The storage schema stores review classification in `metadata_json`; there is no `review_result` column in `reasons/storage.py`.
- `never-reviewed-is-separate-filter` — `list_nodes` accepts `never_reviewed` at `reasons/api.py:3402` and handles it in separate `if never_reviewed:` blocks, not by reading `review_result`.
- `classify-review-result-precedence-unverified` — The body of `_classify_review_result` is absent from the observations, so the exact four-way branch order cannot be verified.
- `review-result-dict-has-verdict-booleans` — `parse_review_response` produces `valid`, `sufficient`, and `necessary` booleans plus optional defeater fields, which are the inputs available to `_classify_review_result`.
