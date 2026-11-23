---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _node_depth in reasons/api.py

# `_node_depth`

## Purpose

`_node_depth` computes a structural depth for a node in the justification graph:

- A **premise** — defined here as a node with no `justifications` — has depth `0`.
- A **missing node** also has depth `0`.
- A **derived node** has depth `1 + max(depth of its antecedents)` over the selected justification(s).

It exists to support filtering/sorting by dependency depth in `list_nodes`, for example `min_depth`, `max_depth`, and `sort="depth"`. It is a private helper, not part of the public API.

The key semantic choice is that depth uses the **maximum** antecedent depth, not the minimum. It also prefers only the currently supporting justification when a valid `supporting_justification` index exists.

---

## Contract

### Preconditions

- `nid` is hashable and normally a string node ID.
- `net` is a `Network`-like object with:
  - `net.nodes`: a mapping supporting `.get(nid)`.
  - Node objects with `.justifications` and `.supporting_justification`.
- Each justification object has `.antecedents`, an iterable of node IDs.
- If `memo` is supplied, it should be a mutable dict used only with the same `net`.
- The graph is expected to be mostly a DAG. Cycles are tolerated only via the provisional cycle guard.

### Postconditions

- Returns an integer `>= 0`.
- If `memo` is supplied, it is mutated to contain depth entries for every visited node ID, including missing IDs.
- For an acyclic graph:
  - Missing node or node with no justifications → `0`.
  - Derived node → `1 + max(depth(a) for a in selected antecedents)`.
- For a cyclic graph, the result is finite but may be root-dependent and approximate.

### Invariants

- A node with no justifications is treated as depth `0`.
- A missing node is treated as depth `0`.
- Only `j.antecedents` are traversed; `j.outlist` and `node.truth_value` are ignored.
- If `supporting_justification` is a valid index into `node.justifications`, only that justification is used.
- Otherwise, all justifications are considered.
- Re-entering a node currently on the recursion stack returns its provisional memo value, initially `0`.

---

## Parameters

### `nid`

The node ID to compute depth for.

Expected type: hashable, usually `str`.

Edge cases:

- If `nid` is not present in `net.nodes`, the function returns `0` and memoizes `memo[nid] = 0`.
- If `nid` is unhashable, e.g. a list, `nid in memo` or `memo[nid]` will raise `TypeError`.

### `net`

The network object containing nodes.

Expected shape:

```python
net.nodes: Mapping[str, Node]
```

Where each node has:

```python
node.justifications: list[Justification]
node.supporting_justification: int | None
```

Each justification has:

```python
justification.antecedents: Iterable[str]
```

The function does not validate this shape. A bad `net` will result in `AttributeError` or `TypeError`.

### `memo`

Optional cache of already-computed depths.

Expected type: `dict[hashable, int] | None`.

Behavior:

- If `None`, a fresh memo is created for this call only.
- If provided, it is mutated and reused across calls.
- If a node ID is already in `memo`, its cached value is returned immediately without inspecting `net`.

Important caller obligation: do not reuse the same `memo` with a different `net` or after the network structure has changed, unless you intentionally want stale depth values.

---

## Return Value

Returns an `int` depth.

Examples for an acyclic graph:

- Missing node → `0`
- Premise with `node.justifications == []` → `0`
- Derived node with one antecedent of depth `0` → `1`
- Derived node with antecedents of depths `1` and `3` → `4`
- Derived node whose selected justification has no antecedents → `1`

The caller must handle:

- Depth `0` does not distinguish between “missing node” and “premise node”.
- In cyclic graphs, depth may not represent a stable topological depth.
- If `memo` is shared, the caller is responsible for keeping it consistent with the current network.

---

## Algorithm

Step by step:

1. If `memo is None`, create a new empty dict.

2. If `nid` is already in `memo`, return the cached value immediately.

3. Look up the node:

   ```python
   node = net.nodes.get(nid)
   ```

4. If the node is missing, or it has no justifications, memoize it as `0` and return `0`.

5. Set a provisional cycle guard:

   ```python
   memo[nid] = 0
   ```

   This means if recursion re-enters `nid` before it finishes, the nested call will see `0` rather than recursing forever.

6. Read `node.supporting_justification`.

7. If it is an in-range index:

   ```python
   si is not None and 0 <= si < len(node.justifications)
   ```

   then use only that one justification.

8. Otherwise, use all justifications.

9. Initialize `max_d = 0`.

10. For each selected justification, for each antecedent ID:

    ```python
    max_d = max(max_d, _node_depth(a, net, memo))
    ```

    `max_d` is the maximum depth among all traversed antecedents.

11. Memoize and return:

    ```python
    memo[nid] = max_d + 1
    return max_d + 1
    ```

The function is recursive and computes the depth of every reachable antecedent unless already memoized.

---

## Side Effects

- Mutates the supplied `memo` dict.
- Uses recursion.
- Does **not** mutate `net`, nodes, justifications, or the database.
- Does **not** perform I/O.
- Does **not** inspect or change truth values.

---

## Error Handling

There is no explicit `try`/`except`.

Potential exceptions:

- `RecursionError` for extremely deep dependency chains.
- `AttributeError` if `net` or a node lacks expected attributes.
- `TypeError` if `nid` is unhashable, `antecedents` is not iterable, `memo` is not a mapping, or indexing `node.justifications[si]` fails because `si` is not an integer index.
- `IndexError` is mostly avoided by the bounds check, but malformed custom containers could still raise.

Notable non-error:

- A missing node is treated as depth `0`; no `KeyError` is raised.
- A missing antecedent is also treated as depth `0`.

Cycle behavior:

- The `memo[nid] = 0` line is a recursive cycle guard, not a full cycle detector.
- In a cyclic graph, re-entering a node returns the provisional `0`.
- The final depth values can depend on which node was visited first and can be lower than the “true” longest cyclic path.
- Example: `A -> B -> A`. Calling `_node_depth("A", net, {})` may produce `A=2, B=1`; calling `_node_depth("B", net, {})` first may produce `B=2, A=1`.

---

## Usage Patterns

`_node_depth` is used inside `list_nodes`, typically like this:

```python
memo = {} if (min_depth is not None or max_depth is not None) else None
d = _node_depth(nid, net, memo)
```

And later:

```python
depth = _node_depth(nid, net, memo) if memo is not None else None
```

Caller obligations:

- For performance across many nodes, pass the same `memo` dict.
- Use the same `net` for the lifetime of that `memo`.
- Do not assume depth `0` means “premise” when the node might be missing.
- Do not expect shortest-path semantics. This function uses maximum antecedent depth.
- Do not expect `outlist` or `truth_value` to influence the result.

`list_nodes` also creates a memo when `sort == "depth"` and computes depths for nodes after filtering.

---

## Assumptions Not Enforced by the Type System

- `net.nodes` is a mapping with `.get`.
- Node objects have truthy/falsy semantics consistent with “exists”; in normal use a node object is truthy.
- `node.justifications` is a list-like object supporting truthiness, `len`, and indexing.
- `node.supporting_justification` is `int | None`. A `bool` technically passes the bounds check because `bool` subclasses `int`; a float could pass comparison but fail indexing.
- Antecedent IDs are valid cache keys, normally strings.
- Missing antecedent IDs are acceptable and silently become depth `0`.
- The graph is mostly acyclic. Cycles are tolerated but not assigned a mathematically stable depth.
- The `memo` cache is not validated against the current network structure.

---

## Dependencies

This function has no imports. It depends only on the object shape of the network:

- `net.nodes`
- `Node.justifications`
- `Node.supporting_justification`
- `Justification.antecedents`

It is conceptually dependent on the semantics defined by `reasons/network.py`, but does not import it.

---

## Topics to Explore

- [function] `reasons/api.py:list_nodes` — Shows how `_node_depth` is used with a shared memo for depth filtering and sorting.
- [file] `reasons/network.py` — Defines `Network`, `Node`, and `supporting_justification` semantics that `_node_depth` assumes.
- [file] `tests/test_api.py` — Contains API-level tests that may exercise depth-based filtering behavior.
- [general] `justification-graph-depth-semantics` — Explore whether depth should use max, min, or supporting-justification-only traversal in cyclic or multi-justification networks.
- [function] `reasons/derive.py:_get_depth` — Compare this helper with the derive/review depth helper used elsewhere in the codebase.

## Beliefs

- `_node-depth-treats-missing-nodes-as-premises` — Missing node IDs are memoized as depth `0` and do not raise `KeyError`.
- `_node-depth-prefers-in-range-supporting-justification` — When `supporting_justification` is a valid index, only that justification’s antecedents are traversed.
- `_node-depth-mutates-shared-memo` — Passing a `memo` dict caches every visited node ID, including missing IDs, for later calls.
- `_node-depth-cycle-guard-is-provisional-zero` — Re-entering a node in a cycle returns the provisional `0`, making depth values order-dependent for cyclic graphs.
- `_node-depth-ignores-outlist-and-truth-values` — Depth traverses only `j.antecedents`; `j.outlist` and `node.truth_value` never affect the result.
