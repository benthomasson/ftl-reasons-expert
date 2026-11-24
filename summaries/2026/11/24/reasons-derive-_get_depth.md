---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _get_depth in reasons/derive.py

## `_get_depth` — `reasons/derive.py`

### Purpose

`_get_depth` computes the depth of a belief node in the reasoning graph. Depth is defined recursively:

- A node that is not a derived node has depth `0` — it is treated as a base premise.
- A derived node has depth `1 + max(depth of its antecedents)`.

The function exists to support prompt construction and filtering in `reasons/derive.py`. It is used to:

- compute the maximum depth in a belief network,
- sort derived conclusions by depth,
- filter beliefs by `min_depth` / `max_depth_filter`.

A key detail: `_get_depth` only follows `antecedents`. It ignores `outlist` dependencies and ignores node truth values.

---

### Contract

#### Preconditions

- `node_id` is hashable, typically a string belief ID.
- `derived` is a mapping from derived node IDs to node data dictionaries.
- Node data in `derived` may contain:
  - `justifications`: a list of justification dictionaries.
  - `supporting_justification`: optional integer index into `justifications`.
- Each justification may contain:
  - `antecedents`: an iterable of antecedent belief IDs.
- `memo`, if provided, is a dictionary used for memoization.
- `nodes` is accepted but is not read at all.

#### Postconditions

- Returns an integer depth.
- For a node not present in `derived`, returns `0`.
- For a node present in `derived`, returns at least `1`.
- Mutates `memo` with computed depths for visited nodes.
- If a cycle exists, prevents infinite recursion by temporarily caching `0` for in-progress nodes.
- Does not guarantee mathematically consistent depths for cyclic graphs; cycle handling is only a recursion guard.

#### Invariants

- `memo[node_id]` is expected to hold the final computed depth once a node finishes processing.
- During recursion, `memo[node_id]` may temporarily be `0` as a cycle guard.
- A node absent from `derived` is always treated as a base node with depth `0`.
- Only antecedents are traversed; outlist nodes do not affect depth.

---

### Parameters

#### `node_id`

The belief ID whose depth should be computed.

Expected type: hashable, usually `str`.

Edge cases:

- If `node_id` is not in `derived`, the function returns `0`.
- If `node_id` is already in `memo`, the cached value is returned immediately.
- If `node_id` appears inside a cycle, the cycle guard may cause it to return provisional values during recursion.

#### `nodes`

The full belief-node mapping.

Expected type: mapping of node ID to node data.

Actual behavior: this parameter is unused. The function never reads `nodes`. This is an important assumption/code smell: all depth-relevant information must be in `derived`.

#### `derived`

A mapping of derived node IDs to node data.

Expected shape:

```python
{
    "some-derived-id": {
        "justifications": [
            {"antecedents": ["a", "b"]},
            {"antecedents": ["c"]},
        ],
        "supporting_justification": 0,  # optional
    },
    ...
}
```

If `supporting_justification` is a valid index, only that justification is used. Otherwise, all justifications are considered.

If a node is missing from `derived`, it is treated as a base premise with depth `0`.

#### `memo`

Optional dictionary used for memoization.

Behavior:

- If `None`, a fresh dictionary is created.
- If provided, it is mutated and reused.
- It may contain cached depths from previous calls.
- Callers can pass a shared `memo` across multiple `_get_depth` calls for the same graph to avoid repeated work.

Important caller obligation: do not reuse the same `memo` across different `derived` graphs unless stale entries are acceptable. Since the function trusts existing `memo` entries, mixing graphs can produce incorrect depths.

---

### Return Value

Returns an integer:

- `0` for base/non-derived nodes.
- `1 + max(antecedent depths)` for derived nodes.
- `1` for a derived node with no antecedents.
- A cached integer if `node_id` is already in `memo`.
- A cycle-guard result of `0` may be returned for a back-edge during recursive traversal.

The caller must handle a finite integer result. The function does not return `None` or error objects.

---

### Algorithm

Step by step:

1. If `memo` is `None`, create a new empty dictionary.
2. If `node_id` is already in `memo`, return the cached value.
3. If `node_id` is not in `derived`, cache `0` and return `0`. This treats the node as a base premise.
4. Set `memo[node_id] = 0` as a temporary cycle guard before recursing.
5. Fetch node data from `derived[node_id]`.
6. Read `justifications` from the node data. If `supporting_justification` is a valid index, restrict traversal to that single justification. Otherwise use all justifications.
7. Initialize `max_d = 0`.
8. For each selected justification:
   - For each antecedent ID:
     - Recursively compute its depth.
     - Update `max_d` to the maximum depth seen.
9. Cache `max_d + 1` as the final depth for `node_id`.
10. Return `max_d + 1`.

Because it uses `max`, the depth is the length of the deepest selected antecedent chain, not a count of justifications.

---

### Side Effects

- Mutates `memo`.
- Does not mutate `nodes` or `derived`.
- Performs no I/O.
- Makes no external API/database calls.
- May raise `RecursionError` on very deep acyclic chains.

Important: if an exception occurs mid-computation, `memo` may be left partially populated, including provisional cycle-guard zeros.

---

### Error Handling

There is no explicit error handling. The function can raise:

- `TypeError` if:
  - `derived[node_id]` is not dictionary-like,
  - `justifications` is not iterable or sized,
  - `supporting_justification` is not an integer but passes the comparison checks,
  - an antecedent list is not iterable.
- `IndexError` / `TypeError` for invalid indexing behavior in unusual data shapes.
- `RecursionError` for very deep acyclic chains.
- Potential `AttributeError` if `node_data` or a justification is not a dictionary-like object with `.get`.

The code assumes:

- `justifications` is a list if present.
- `antecedents` is an iterable if present.
- `supporting_justification` is either `None` or a valid integer-like index.
- Antecedent IDs are hashable.

Unknown antecedent IDs are not rejected. They are simply treated as base nodes because they are absent from `derived`.

---

### Usage Patterns

Typical call sites:

#### 1. Sorting derived conclusions in `_build_derived_section`

```python
memo = {}
for k in sorted(derived, key=lambda x: -_get_depth(x, nodes, derived, memo)):
    depth = _get_depth(k, nodes, derived, memo)
```

The same `memo` is reused across calls so each derived node is computed once.

#### 2. Computing max depth in `build_prompt`

```python
all_derived = {k: v for k, v in nodes.items()
               if v.get("justifications") and len(v["justifications"]) > 0}
memo = {}
max_depth = max((_get_depth(k, nodes, all_derived, memo) for k in all_derived), default=0)
```

Here `nodes` is passed but unused. The real input is `all_derived`.

#### 3. Filtering by depth

`build_prompt` also uses `_get_depth` when applying `min_depth` and `max_depth_filter`.

Caller obligations:

- Pass a `derived` mapping that includes all nodes whose depth should propagate.
- Reuse `memo` only for the same graph.
- Do not expect outlist semantics to affect depth.
- Be aware that cycles produce finite but potentially order-dependent depths.

---

### Dependencies

Internal dependencies:

- None directly inside `_get_depth`.
- It relies on the data shape produced by the surrounding `reasons/derive.py` pipeline, especially `justifications`, `antecedents`, and `supporting_justification`.

External dependencies:

- Python built-ins only.
- No imports, no I/O, no database access.

---

## Topics to Explore

- [file] `reasons/derive.py` — Full derive pipeline: prompt building, proposal parsing, validation, and applying new derived beliefs.
- [function] `reasons/derive.py:_build_derived_section` — Uses `_get_depth` to sort and display derived conclusions by depth.
- [function] `reasons/derive.py:build_prompt` — Computes `max_depth` and applies depth-based filtering using `_get_depth`.
- [file] `tests/test_derive_budget.py` — Likely covers budget/sampling behavior and depth-related prompt construction.
- [general] `reason-maintenance-depth-semantics` — Explore how depth is defined and used in the broader RMS, especially why `outlist` dependencies are excluded from `_get_depth`.

---

## Beliefs

- `_get-depth-ignores-nodes-parameter` — The `nodes` parameter is accepted but never read; depth is computed solely from `derived`.
- `_get-depth-respects-supporting-justification` — If `supporting_justification` is a valid index, only that justification’s antecedents are traversed.
- `_get-depth-ignores-outlist` — Depth considers only `antecedents`; `outlist` dependencies do not affect the computed depth.
- `_get-depth-treats-non-derived-as-base` — Any node ID not present in `derived` is treated as depth `0`.
- `_get-depth-cycle-guard-can-underestimate` — Setting `memo[node_id] = 0` before recursion prevents infinite loops but can produce order-dependent depths for cyclic graphs.
