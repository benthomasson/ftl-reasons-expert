---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: format_belief_for_review in reasons/review.py

## Overview

`format_belief_for_review` is a pure formatting helper in `reasons/review.py`. It serializes one TMS node into a compact, text representation suitable for inclusion in an LLM review prompt. The output includes the node ID, its claim text, and each justification’s antecedents and outlist items with their referenced node texts.

It does **not** validate reasoning. It only prepares text for later LLM-based review.

## Contract

### Preconditions

- `nodes` should be a dict-like mapping from node ID to a node dictionary.
- `node_id` should be a key into `nodes`, or absent.
- A node’s expected fields are:
  - `text`: claim text
  - `justifications`: list of justification dicts
- Each justification dict is expected to optionally contain:
  - `antecedents`: list of node IDs
  - `outlist`: list of node IDs that must be `OUT`
  - `label`: short label/string

These expectations are conventions, not enforced by types or runtime validation.

### Postconditions

- If `nodes.get(node_id)` is missing or falsy, the function returns `""`.
- Otherwise it returns a newline-joined string containing:
  - `### <node_id>`
  - `Claim: <node text>`
  - one block per justification, preserving list order
  - antecedent lines when present
  - outlist lines when present
  - justification label when present
- Missing referenced antecedent/outlist nodes do not raise. They are rendered as:
  - `- <id>: (not found in network)`
- The input `nodes` structure is not mutated.
- There is no trailing newline and no blank line between justifications.

### Invariants

- Ordering of `justifications`, `antecedents`, and `outlist` is preserved.
- Justification headers are only emitted when there is more than one justification:
  - `Justification 1/2:`
  - `Justification 2/2:`
- A single justification is emitted without a `Justification 1/1:` header.
- Referenced node text is pulled from the top-level `nodes` mapping, not from nested justification data.

### Assumptions not enforced by types

The function assumes:

- `nodes` has a `.get` method.
- Node values are dict-like when truthy.
- Justification entries are dict-like.
- `justifications` is iterable and, when present, not `None`.
- `antecedents` and `outlist`, when truthy, are iterable collections of hashable node IDs.
- Looking up referenced IDs with `nodes.get(...)` is valid.
- Text values are stringifiable.

Violating these assumptions can raise `AttributeError` or `TypeError`.

## Parameters

### `node_id`

The ID of the node to format. It is normally a string. It is used directly in the heading and passed to `nodes.get(node_id)`.

Edge cases:

- If the key is absent, the function returns `""`.
- If the key maps to a falsy value, such as `None` or `{}`, the function also returns `""`.
- If the key is unhashable, `nodes.get(node_id)` can raise `TypeError`.

### `nodes`

Mapping of node ID to node data, typically produced by `export_network()`.

Expected node shape:

```python
{
    "text": "...",
    "justifications": [
        {
            "antecedents": ["ant-id-1"],
            "outlist": ["out-id-1"],
            "label": "...",
        }
    ],
}
```

Edge cases:

- If `nodes` is `None`, `.get` raises `AttributeError`.
- If referenced nodes are missing, placeholders are emitted.
- If a referenced node exists but is falsy, such as `{}`, it is treated as not found because the code checks `if ant_node:` / `if out_node:`.
- If `justifications` exists but is `None`, iteration raises `TypeError`.
- If a justification entry is not a dict, `.get` raises `AttributeError`.

## Return Value

Returns a string.

- Returns `""` when `node_id` is not found or maps to a falsy node.
- Otherwise returns formatted text like:

```text
### belief-1
Claim: A implies B.
Antecedents:
- premise-1: A is true.
Unless (must be OUT):
- exception-1: A known exception.
Label: default
```

For multiple justifications, it emits justification headers:

```text
### belief-1
Claim: A implies B.
Justification 1/2:
Antecedents:
- premise-1: A is true.
Justification 2/2:
Antecedents:
- premise-2: B follows from C.
```

Callers must handle the empty-string case. In the main caller `review_beliefs`, beliefs are pre-filtered to existing nodes with justifications, so `""` is normally not expected there.

## Algorithm

1. Look up `node = nodes.get(node_id)`.
2. If `node` is falsy, return `""`.
3. Start the output lines with `### {node_id}`.
4. Append `Claim: {node.get("text", "")}`.
5. Read `justs = node.get("justifications", [])`.
6. Iterate through each justification with a 1-based index.
7. For each justification:
   - Read `antecedents`, defaulting to `[]`.
   - Read `outlist`, defaulting to `[]`.
   - Read `label`, defaulting to `""`.
   - If there is more than one justification, append `Justification i/n:`.
   - If antecedents are present:
     - append `Antecedents:`
     - for each antecedent ID, look it up in `nodes`
     - append `- id: text` if the referenced node is truthy
     - append `- id: (not found in network)` otherwise
   - If outlist is present:
     - append `Unless (must be OUT):`
     - format each outlist entry the same way as antecedents
   - If `label` is truthy, append `Label: {label}`.
8. Join all lines with `"\n"` and return the result.

## Side Effects

None.

The function:

- does not mutate `nodes` or the node dictionaries
- does not write to files
- does not print
- does not call an LLM
- does not update global state

It only reads data and builds a new list of strings.

## Error Handling

There is no explicit `try`/`except` inside `format_belief_for_review`.

Handled gracefully:

- Missing top-level node: returns `""`
- Missing referenced antecedent node: emits `(not found in network)`
- Missing referenced outlist node: emits `(not found in network)`

Not handled gracefully:

- `nodes` is `None`: `AttributeError`
- node is truthy but not dict-like: `AttributeError`
- `justifications` is `None`: `TypeError`
- a justification entry is not dict-like: `AttributeError`
- `antecedents` or `outlist` is truthy but not iterable: `TypeError`
- referenced node exists but is truthy and not dict-like: `AttributeError`

Important: in `review_beliefs`, formatting happens before the `try` block around the LLM call:

```python
beliefs_text = "\n\n".join(
    format_belief_for_review(nid, nodes) for nid in batch
)
prompt = REVIEW_PROMPT.format(beliefs=beliefs_text)

try:
    response = invoke_model(...)
```

So if formatting raises due to malformed node data, that exception is not caught by the batch-level LLM error handling and will abort the review call unless caught by a higher-level caller.

## Usage Patterns

Primary usage is inside `review_beliefs`:

```python
beliefs_text = "\n\n".join(
    format_belief_for_review(nid, nodes) for nid in batch
)
prompt = REVIEW_PROMPT.format(beliefs=beliefs_text)
```

The caller is responsible for:

- passing the full network in `nodes`, so antecedent/outlist IDs can be resolved to text
- ensuring each node has the expected schema
- deciding which beliefs to review
- handling the empty string if calling this helper directly on an unknown node

`review_beliefs` normally filters beliefs to existing nodes with non-empty `justifications`. When `belief_ids` is not provided, it also filters to `truth_value == "IN"` nodes. If `belief_ids` is provided explicitly, `review_beliefs` still requires the node to exist and have justifications, but it does not re-check `truth_value`.

The function is also useful directly in tests to assert prompt formatting behavior.

## Dependencies

This function depends only on Python built-ins and the implicit TMS node schema.

It relies on:

- `.get` on mappings
- string formatting
- iteration over justification/antecedent/outlist lists

It does **not** depend on `invoke_model`, `json`, or `sys`, even though those are imported by the surrounding module.

Relevant schema fields come from TMS/network export data:

- `node["text"]`
- `node["justifications"]`
- `justification["antecedents"]`
- `justification["outlist"]`
- `justification["label"]`

## Topics to Explore

- [file] `reasons/review.py` — Full review pipeline: filtering beliefs, batching, prompting, parsing, and classification.
- [function] `reasons/review.py:review_beliefs` — Shows how formatted belief strings are batched and passed to the LLM reviewer.
- [function] `reasons/review.py:parse_review_response` — Explains how the LLM’s JSON review output is extracted and normalized.
- [file] `tests/test_review_justifications.py` — Likely exercises multiple-justification formatting and label/antecedent rendering.
- [general] `tms-justification-schema` — Useful for understanding the semantics of `antecedents`, `outlist`, and `label` in this codebase.

## Beliefs

- `format-belief-for-review-returns-empty-string-for-missing-node` — If `nodes.get(node_id)` is falsy, the function returns `""` instead of raising.
- `format-belief-for-review-includes-referenced-node-text-or-placeholder` — For each antecedent or outlist ID, it emits the referenced node’s `text` when the lookup is truthy, otherwise `(not found in network)`.
- `format-belief-for-review-does-not-mutate-input` — The function only reads from `nodes` and builds a new list of strings.
- `format-belief-for-review-omits-justification-header-for-single-justification` — It emits `Justification i/n:` only when `len(justs) > 1`.
- `format-belief-for-review-assumes-dict-shaped-node-and-justification-data` — It calls `.get` and iterates fields without validation, so malformed non-dict or non-iterable values can raise `AttributeError` or `TypeError`.
