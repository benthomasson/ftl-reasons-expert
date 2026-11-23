---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _cascade_depth in reasons/api.py

# `_cascade_depth`

## Purpose

`_cascade_depth` is a private BFS helper in `reasons/api.py`. It measures the shortest directed distance from a root node (`retracted_id`) to a target node (`target_id`) by walking the `dependents` edges of the in-memory `Network`.

Despite the docstring saying “shortest justification path,” the implementation is purely graph traversal: it does not inspect justifications, edge types, truth values, or whether an intermediate node actually changed truth value. It is used by `what_if_retract`, `what_if_assert`, and `what_if_supersede` to attach a `depth` to each affected node and then sort affected nodes by `(depth, id)`.

## Contract

### Preconditions

- `net` is a network-like object with a `.nodes` mapping of node ID to node objects.
- Each node object has a `.dependents` iterable containing dependent node IDs.
- `retracted_id` must exist as a key in `net.nodes`; otherwise the first queue pop raises `KeyError`.
- Every dependent ID encountered must either:
  - equal `target_id`, or
  - exist in `net.nodes`.
  If a non-target dependent ID is missing from `net.nodes`, it is enqueued and later causes `KeyError` when looked up.
- `target_id` is expected to be a string, but it is not validated and does not have to exist in `net.nodes`.

### Postconditions

- Returns a non-negative integer.
- If `target_id` is reachable from `retracted_id` through zero or more `dependents` edges, returns the number of edges in the shortest path.
- If `target_id == retracted_id`, returns `0`.
- If no path exists, returns `0`.
- The function does not mutate `net`, node truth values, justifications, or any database state.

### Important invariant

The BFS uses a FIFO queue and marks nodes visited at discovery time. In an unweighted graph, this guarantees that the first time `target_id` is found, the returned depth is the shortest path length in terms of number of `dependents` edges.

## Parameters

### `net`

A `Network`-like object, not typed in the signature. It must expose:

```python
net.nodes[node_id].dependents
```

`dependents` is assumed to be an iterable of node IDs. In this codebase it appears to be a reverse dependency index: if `A` depends on `B`, then `B.dependents` contains `A`. The index appears to include both antecedent dependencies and outlist dependencies, based on other parts of `api.py` that rebuild dependents from both `j.antecedents` and `j.outlist`.

### `target_id: str`

The node whose cascade depth is being measured.

Edge cases:

- If `target_id` is the same as `retracted_id`, the function returns `0`.
- If `target_id` does not exist in `net.nodes`, the function may still return `0` or a positive depth if `target_id` appears in some dependent set. It does not validate target existence.
- If `target_id` is reached as a dependent, the function returns before checking whether the target node actually exists.

### `retracted_id: str`

The starting node for BFS.

Despite the name, callers use this for more than retraction:

- `what_if_retract` passes the retracted node.
- `what_if_assert` passes the asserted node.
- `what_if_supersede` passes the old node ID.

Edge cases:

- If `retracted_id` is absent from `net.nodes`, `KeyError` is raised immediately when the initial queue entry is popped.
- If `retracted_id` has cycles back to itself, the initial `visited = {retracted_id}` prevents reprocessing.

## Return Value

Returns an `int`:

- `0` if `target_id == retracted_id`.
- `0` if no path from `retracted_id` to `target_id` exists through `dependents`.
- `1` if `target_id` is a direct dependent of `retracted_id`.
- `N` if the shortest path has `N` dependent edges.

The return value conflates two cases:

1. target is the root itself, and
2. target is unreachable.

Callers must interpret `0` carefully. In current usage, the root node is skipped before calling `_cascade_depth`, so `0` mostly means “unreachable” or “no depth found.” However, the sort logic does not distinguish this from a legitimate depth-0 root.

## Algorithm

Step by step:

1. Import `deque` from `collections`.
2. Initialize `visited` with `retracted_id`.
3. Initialize a FIFO queue with `(retracted_id, 0)`.
4. While the queue is non-empty:
   1. Pop the leftmost `(current_id, depth)`.
   2. Look up `current = net.nodes[current_id]`.
   3. For each `dep_id` in `current.dependents`:
      - If `dep_id` is already visited, skip it.
      - If `dep_id == target_id`, return `depth + 1`.
      - Otherwise mark `dep_id` visited and enqueue `(dep_id, depth + 1)`.
5. If the queue empties without finding `target_id`, return `0`.

This is a standard BFS over an unweighted directed graph. Because `visited` is updated when a node is discovered, not when it is popped, each node is enqueued at most once.

## Side Effects

No persistent side effects.

The function:

- reads from `net.nodes`,
- reads each node’s `.dependents`,
- uses a local `visited` set and `deque`,
- imports `deque` locally.

It does not write to `net`, does not touch SQLite, does not call `Storage`, and does not publish events.

## Error Handling

The function has no explicit `try`/`except`.

Possible exceptions:

- `KeyError` if `retracted_id` is not in `net.nodes`.
- `KeyError` if a dependent ID is enqueued and later looked up in `net.nodes` but is missing.
- `AttributeError` if `net` lacks `.nodes`, or a node lacks `.dependents`.
- `TypeError` if `dependents` is not iterable.

Missing `target_id` does not necessarily raise. If the target is never encountered, the function returns `0`. If the target string appears in a `dependents` set but has no corresponding node entry, the function can return a positive depth without ever validating that the target node exists.

## Usage Patterns

The helper is used only inside `what_if_*` simulation functions in `reasons/api.py`.

Typical call:

```python
info = {
    "id": nid,
    "text": n.text,
    "depth": _cascade_depth(net, nid, node_id),
    "dependents": len(n.dependents),
}
```

Then affected nodes are sorted:

```python
retracted.sort(key=lambda c: (c["depth"], c["id"]))
restored.sort(key=lambda c: (c["depth"], c["id"]))
```

Caller obligations:

- Pass an in-memory `Network` whose `dependents` reverse index is populated.
- Ensure `retracted_id` exists.
- Prefer not to call it for the root node, because the result will be `0` and may sort before actually affected nodes.
- Treat `0` as ambiguous between “root” and “unreachable.”

## Dependencies

- `collections.deque` for BFS.
- The `Network` node model, specifically the `.dependents` reverse dependency index.
- Indirectly, the callers in `reasons/api.py`:
  - `what_if_retract`
  - `what_if_assert`
  - `what_if_supersede`

## Assumptions Not Enforced by Type System

- `retracted_id` exists in `net.nodes`.
- Every dependent ID references an existing node unless it is the target.
- `dependents` is already maintained as a reverse index, including both antecedent and outlist edges.
- The graph may contain cycles; BFS handles them with `visited`.
- `target_id` may be missing; the function does not validate it.
- The shortest path through `dependents` is treated as the “cascade depth,” even if intermediate nodes did not actually change truth value or the edge belongs to an inactive justification.
- The function does not filter by truth value, justification type, label, source, or whether an edge is currently supporting.

## Topics to Explore

- [function] `reasons/api.py:what_if_retract` — Primary caller that uses `_cascade_depth` to report and sort cascade effects after simulated retraction.
- [function] `reasons/api.py:what_if_assert` — Shows that the “retracted_id” parameter is really a generic root node and can represent an asserted node.
- [function] `reasons/api.py:what_if_supersede` — Demonstrates depth calculation after a simulated supersession, using `old_id` as the BFS root.
- [file] `reasons/network.py` — Defines `Node.dependents` and the reverse dependency index that `_cascade_depth` traverses.
- [file] `tests/test_api.py` — Likely contains behavioral coverage for `what_if_*` output and depth sorting.

## Beliefs

- `_cascade-depth-returns-shortest-dependent-path` — For a reachable target other than the start node, `_cascade_depth` returns the number of `dependents` edges in the shortest path from `retracted_id` to `target_id`.
- `_cascade-depth-returns-zero-for-unreachable-target` — When no dependent path to `target_id` exists, or when `target_id == retracted_id`, `_cascade_depth` returns `0`.
- `_cascade-depth-uses-dependents-only` — `_cascade_depth` traverses only `net.nodes[current_id].dependents` and ignores justification types, outlists, truth values, and labels.
- `_cascade-depth-requires-retracted-node` — If `retracted_id` is absent from `net.nodes`, `_cascade_depth` raises `KeyError` on the first queue pop.
- `_cascade-depth-does-not-validate-target` — A missing `target_id` does not necessarily raise; it returns `0` if unreachable or can return a positive depth if the ID appears in a `dependents` set without a node entry.
