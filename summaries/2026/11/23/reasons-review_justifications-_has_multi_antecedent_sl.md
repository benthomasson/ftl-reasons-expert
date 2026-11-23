---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _has_multi_antecedent_sl in reasons/review_justifications.py

## `_has_multi_antecedent_sl`

### Purpose

`_has_multi_antecedent_sl` is a small predicate used to decide whether a TMS node is a candidate for LLM review of its SL justification classification.

Specifically, it answers:

> Does this node have at least one SL justification whose antecedent list has at least `min_antecedents` entries?

It exists to filter beliefs before sending them to an LLM in `review_justifications()`. The surrounding review flow is concerned with justifications that have multiple antecedents, because those are the cases where the distinction between ALL/conjunctive and ANY/disjunctive support matters. Despite the “multi” in the name, this function does not hard-code “multiple” as `>= 2`; the threshold is controlled by the `min_antecedents` parameter.

---

### Contract

#### Preconditions

The function implicitly assumes:

- `node` is a mapping-like object with a `.get()` method, typically a `dict`.
- If `node["justifications"]` exists, it is an iterable of mapping-like objects.
- Each justification object may have:
  - a `"type"` value, expected to be a string such as `"SL"`;
  - an `"antecedents"` value, expected to be a sized collection such as a list.
- `min_antecedents` is comparable to the result of `len(...)`, normally an integer.

None of these assumptions are enforced by the function itself.

#### Postconditions

- Returns `True` iff there exists at least one justification `j` in `node.get("justifications", [])` such that:
  - `j.get("type") == "SL"`, and
  - `len(j.get("antecedents", [])) >= min_antecedents`.
- Returns `False` if no such justification exists, including when the node has no `"justifications"` key or an empty justification list.
- The function short-circuits: it returns `True` as soon as it finds the first qualifying SL justification.

#### Invariants

- It is a pure predicate: it does not mutate `node` or any justification.
- It does not inspect `truth_value`.
- It does not validate antecedent IDs against `nodes`.
- It does not check whether the qualifying SL justification is the only justification, or whether other justifications have different forms.
- The `"type"` comparison is exact and case-sensitive.

---

### Parameters

#### `node`

Expected type: `dict`-like node data from `export_network()`.

The function reads `node.get("justifications", [])`. Typical shape:

```python
{
    "truth_value": "IN",
    "justifications": [
        {
            "type": "SL",
            "antecedents": ["a", "b", "c"],
        }
    ],
}
```

Edge cases:

- If `"justifications"` is missing, `.get(..., [])` supplies an empty list and the function returns `False`.
- If `"justifications"` exists but is `None`, the `for` loop raises `TypeError`.
- If `node` itself is not mapping-like, `node.get(...)` raises `AttributeError`.
- If an item inside `"justifications"` is not mapping-like, `j.get(...)` raises `AttributeError`.

#### `min_antecedents`

Expected type: integer-like value, normally `>= 1`.

This is the minimum required length of the antecedent list. The caller `review_justifications()` defaults it to `2`.

Edge cases:

- If `min_antecedents == 0`, any SL justification qualifies, even one with an empty or missing antecedent list.
- If `min_antecedents` is negative, any SL justification also qualifies because every non-negative length is `>=` a negative number.
- If `min_antecedents` is `None` or a non-comparable type such as a string, the comparison can raise `TypeError`.
- If a justification has no `"antecedents"` key, `j.get("antecedents", [])` treats it as an empty list. Therefore, with `min_antecedents > 0`, it does not qualify; with `min_antecedents == 0`, it does.

---

### Return Value

Returns a `bool`:

- `True` if at least one `"SL"` justification has `len(antecedents) >= min_antecedents`.
- `False` otherwise.

The caller must treat this as only a filtering decision. It does not mean:

- the node’s truth value is `"IN"`;
- the node is definitely misclassified;
- all SR justifications have enough antecedents;
- the matching justification is the one the LLM will explicitly review.

In `review_justifications()`, the caller combines this predicate with a `truth_value == "IN"` check.

---

### Algorithm

Step by step:

1. Iterate over `node.get("justifications", [])`.
2. For each justification `j`:
   - Read `j.get("type")`.
   - If it is exactly `"SL"`, read `j.get("antecedents", [])`.
   - Compute its length.
   - Compare that length to `min_antecedents`.
3. If both conditions hold, immediately return `True`.
4. If the loop finishes without a match, return `False`.

Pseudocode:

```python
for j in node.get("justifications", []):
    if j.get("type") == "SL":
        antecedents = j.get("antecedents", [])
        if len(antecedents) >= min_antecedents:
            return True
return False
```

The short-circuit behavior makes the best-case cost constant and the worst-case cost proportional to the number of justifications on the node.

---

### Side Effects

None.

The function:

- does not mutate `node`;
- does not mutate justifications;
- does not read or write files;
- does not print;
- does not call the LLM;
- does not access global state.

It can still raise exceptions if the input structure violates the implicit assumptions, but it has no intentional side effects.

---

### Error Handling

The function does not catch exceptions. Errors propagate to the caller.

Possible failures include:

- `AttributeError` if `node` or a justification element lacks `.get()`;
- `TypeError` if `"justifications"` is not iterable, e.g. `None`;
- `TypeError` if `"antecedents"` is `None` or otherwise has no valid length;
- `TypeError` if `min_antecedents` cannot be compared with an integer length.

There is no defensive validation that:

- `"antecedents"` is a list;
- each antecedent ID exists in `nodes`;
- `"type"` is a string;
- `min_antecedents` is non-negative.

The surrounding review code assumes exported network nodes have the expected shape.

---

### Usage Patterns

This is a private helper, indicated by the leading underscore.

It is used inside `review_justifications()` as part of candidate selection:

```python
belief_ids = sorted(
    nid for nid, node in nodes.items()
    if node.get("truth_value") == "IN"
    and _has_multi_antecedent_sl(node, min_antecedents)
)
```

When explicit `belief_ids` are supplied, the same predicate is still applied as a filter:

```python
belief_ids = [
    nid for nid in belief_ids
    if nid in nodes
    and nodes[nid].get("truth_value") == "IN"
    and _has_multi_antecedent_sl(nodes[nid], min_antecedents)
]
```

Caller obligations:

- Pass node data with the expected `"justifications"` structure.
- Apply any truth-value filtering separately if needed.
- Ensure `min_antecedents` is a sensible integer.
- Do not rely on this helper to validate antecedent IDs or justification completeness.

A subtle usage assumption: a node qualifies if **any** SL justification meets the threshold. The later LLM review receives the belief/node, not necessarily only the single matching justification. So a node with mixed justification shapes can still enter review.

---

### Dependencies

This function has no imports and depends only on Python built-ins.

It relies on the broader data contract from the TMS export:

- nodes contain `"justifications"`;
- justifications contain `"type"` and `"antecedents"`;
- SL justifications are represented by the exact string `"SL"`.

It is consumed by:

- `reasons/review_justifications.py:review_justifications`
- indirectly, via candidate selection for `_process_batch()` and `invoke_model()`

---

## Topics to Explore

- [function] `reasons/review_justifications.py:review_justifications` — Shows how this predicate filters IN beliefs before batching them for LLM review.
- [file] `reasons/review_justifications.py` — Contains the full ALL/ANY/MIXED review flow, prompt construction, batching, and parsing.
- [function] `reasons/review.py:format_belief_for_review` — Determines how candidate nodes and their justifications are serialized into the LLM prompt.
- [file] `tests/test_review_justifications.py` — Verifies expected filtering, parsing, and review behavior around this predicate.
- [general] `truth-maintenance-system-justification-types` — Background on SL justifications and ALL/ANY semantics in the TMS model.

---

## Beliefs

- `_has_multi_antecedent_sl-is-pure-predicate` — The function performs no mutation, I/O, or external calls and returns only a boolean.
- `qualifies-if-any-sl-justification-meets-threshold` — It returns `True` as soon as one justification has type `"SL"` and enough antecedents.
- `missing-antecedents-default-to-empty-list` — A justification without an `"antecedents"` key is treated as having zero antecedents.
- `type-match-is-exact-and-case-sensitive` — Only justifications whose `"type"` value is exactly `"SL"` are considered.
- `caller-must-enforce-in-truth-value` — The predicate does not check `truth_value == "IN"`; `review_justifications` applies that filter separately.
