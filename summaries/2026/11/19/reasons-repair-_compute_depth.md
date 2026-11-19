---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _compute_depth in reasons/repair.py

## Purpose

`_compute_depth` computes the structural derivation depth of a belief in the Truth Maintenance System dependency graph.

- Premises / nodes without justifications have depth `0`.
- A derived node has depth `1 + max(depth of all antecedent nodes)`.
- It is used by `repair_beliefs` to populate `depth` in the triage prompt, helping the LLM decide whether an invalid belief is shallow enough to repair or deep/broken enough to abandon.

It exists mainly as a memoized recursive helper so the same dependency depth is not recomputed repeatedly across multiple invalid beliefs in the same repair batch.

## Contract

### Preconditions

- `node_id` should be a hashable identifier, typically a string.
- `nodes` should be a mapping-like object supporting `.get(node_id)`, typically `dict[str, dict]`.
- Each node should be a mapping-like object supporting `.get("justifications")`.
- If present and truthy, `node["justifications"]` should be an iterable of justification mappings.
- Each justification should support `.get("antecedents", [])`; antecedents should be an iterable of node IDs.
- `memo`, if provided, should be a mutable mapping with node IDs as keys and integer depths as values.
- For meaningful “depth” semantics, the dependency graph should be acyclic.

### Postconditions

- Returns a non-negative integer for normally-shaped input.
- Populates `memo` with computed depths for `node_id` and recursively reached nodes.
- Does not mutate `nodes`.
- For an acyclic graph, the returned value is the correct maximum derivation depth.
- For a cyclic graph, recursion is prevented but the returned depth is not a mathematically meaningful graph depth.

### Invariants / assumptions not enforced by the type system

- A missing node or a node with absent/empty justifications is treated as a premise, returning `0`.
- A non-empty justification list with no antecedents returns depth `1`, because `max_d` remains `0` and the function returns `max_d + 1`.
- `outlist`, `unless`, truth values, and TMS semantics are ignored. Depth is purely structural.
- Missing antecedent IDs are treated as depth `0`, indistinguishable from actual premises.
- The placeholder `memo[node_id] = 0` before recursion acts as a cycle guard, but it means cyclic graphs produce traversal-order-dependent results.

## Parameters

- `node_id`: Identifier of the node whose derivation depth should be computed. Typically a string. If not present in `nodes`, the function returns `0`.
- `nodes`: Mapping of node ID to node dictionaries. Each node is expected to contain an optional `"justifications"` key. Each justification may contain an `"antecedents"` list.
- `memo`: Optional mutable cache mapping node IDs to depths. If omitted or `None`, a fresh dictionary is created for that call. If provided, it is reused and mutated across recursive calls and possibly across multiple top-level calls sharing the same graph snapshot.

## Return Value

Returns an integer depth:

- `0` if the node is missing.
- `0` if the node has no truthy `"justifications"`.
- Otherwise, `1 + max(depth of each antecedent)` across all justifications.
- If a node has non-empty justifications but all antecedent lists are empty, it returns `1`.

The caller must handle the possibility of exceptions for malformed graph structures, and should not rely on depth values being meaningful for cyclic graphs.

## Algorithm

1. If `memo` is `None`, initialize it to `{}`.
2. If `node_id` is already in `memo`, return the cached value.
3. Look up `node = nodes.get(node_id)`.
4. If `node` is missing or has no truthy `"justifications"`:
   - Cache `memo[node_id] = 0`.
   - Return `0`.
5. Otherwise, set `memo[node_id] = 0` as an in-progress placeholder. This prevents infinite recursion if the graph contains a cycle back to this node.
6. Initialize `max_d = 0`.
7. For each justification in `node["justifications"]`:
   - For each antecedent `a` in `j.get("antecedents", [])`:
     - Recursively compute `_compute_depth(a, nodes, memo)`.
     - Update `max_d = max(max_d, antecedent_depth)`.
8. Set `memo[node_id] = max_d + 1`.
9. Return `max_d + 1`.

## Side Effects

- Mutates the provided `memo` dictionary, or allocates and mutates a local one if none was passed.
- Uses recursion, so deep dependency chains can consume recursion stack.
- Performs no I/O, network calls, database writes, or mutations to `nodes`.

## Error Handling

`_compute_depth` does not catch exceptions. It can raise:

- `AttributeError` if `nodes` does not support `.get`, or if a node/justification does not support `.get`.
- `TypeError` if node IDs are unhashable, if `justifications` is truthy but not iterable, or if malformed types are used in arithmetic/comparison.
- `RecursionError` for sufficiently deep acyclic chains.
- `KeyError` is unlikely under normal dict behavior because the function uses `.get`, but malformed mapping-like objects could still produce unexpected failures.

In `repair_beliefs`, the call is inside a broad per-belief `try/except Exception` block, so exceptions from `_compute_depth` are captured into that belief’s `result["error"]`. The helper itself does not swallow errors.

## Usage Patterns

Typical usage is inside `repair_beliefs`:

```python
depth_memo = {}
for r in invalid:
    belief_id = r["id"]
    depth = _compute_depth(belief_id, nodes, depth_memo)
```

Important caller obligations:

- Reuse the same `memo` only while the `nodes` graph snapshot is unchanged.
- Use a fresh memo if the graph is modified, because stale cached depths would otherwise be returned.
- Treat the result as structural depth only, not as a measure of TMS validity or repair difficulty.
- Do not pass cyclic graphs if exact depth semantics matter.

## Dependencies

- No imports or external modules.
- Relies only on Python dictionary/mapping operations and recursion.
- Indirectly supports `repair_beliefs` and the triage prompt in `reasons/repair.py`.

## Topics to Explore

- [file] `reasons/repair.py` — See how `_compute_depth` feeds into the triage prompt and repair orchestration.
- [function] `reasons/repair.py:repair_beliefs` — Shows how `depth_memo` is shared across invalid beliefs and how exceptions are captured.
- [function] `reasons/repair.py:triage_belief` — Consumes the computed depth when selecting `search_and_link`, `soften`, `abandon`, or `research`.
- [general] `truth-maintenance-system-graph-structure` — Important for understanding justifications, antecedents, outlists, and why depth is structural.
- [file] `tests/test_repair.py` — Likely contains expectations for repair behavior and depth-related helper usage.

## Beliefs

- `compute-depth-returns-zero-for-missing-nodes` — A missing `node_id` and a node with absent or empty `"justifications"` both return depth `0`.
- `compute-depth-uses-max-across-all-justifications` — For a node with multiple justifications, the returned depth is `1 +` the maximum antecedent depth across all justifications.
- `compute-depth-mutates-shared-memo` — The optional `memo` mapping is populated with computed values and reused across recursive and top-level calls.
- `compute-depth-cycle-guard-is-memo-placeholder` — An in-progress node is temporarily memoized as `0`, preventing infinite recursion but producing traversal-dependent depths for cyclic graphs.
- `compute-depth-propagates-malformed-structure-errors` — Malformed node or justification structures can raise `AttributeError`, `TypeError`, or `RecursionError`; the helper itself does not catch them.
