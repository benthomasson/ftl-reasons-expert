---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: trace_assumptions in reasons/network.py

## Purpose

`trace_assumptions` performs a backward traversal of the justification graph to collect the structural roots of a node’s support. A root is defined here as any reachable node whose `justifications` list is empty — i.e. a premise. These are the assumptions that the conclusion ultimately rests on.

It exists primarily to support dependency-directed backtracking: `find_culprits` uses it to determine which premises underlie a contradictory set of `IN` nodes, so that a low-entrenchment premise can be retracted to resolve a nogood.

Important nuance: this method traces the **syntactic antecedent graph**, not the currently active or valid justification chain. It walks every justification’s antecedents, regardless of truth values or validity.

## Contract

### Preconditions

- `node_id` should exist in `self.nodes`.
- Nodes are expected to have a `justifications` attribute.
- Each justification is expected to have an iterable `antecedents` attribute containing node IDs.

### Postconditions

- Returns a `list[str]` of premise IDs reachable from `node_id` by following `Justification.antecedents` edges.
- A premise is defined structurally as `not node.justifications`.
- The starting node itself is included if it has no justifications.
- The returned list contains no duplicates.
- The returned list is in DFS discovery order, not sorted and not necessarily stable across graph mutations.
- The network is not modified.

### Invariants / Implicit Assumptions

- `visited` prevents cycles and duplicate traversal.
- Missing antecedent IDs are silently skipped.
- Outlist nodes are never traversed.
- Justification validity is not checked, so premises from invalid or alternative justifications are still included.
- Truth values are not checked, so a retracted premise (`truth_value == "OUT"` but still zero justifications) can be returned.
- A node with an empty-antecedent justification is **not** treated as a premise. Only `not node.justifications` counts.
- The docstring says “set”, but the actual return type is `list`.

## Parameters

- `node_id: str`
  - The node whose assumptions should be traced.
  - Must be a key in `self.nodes`; otherwise `KeyError` is raised.
  - Can point to a premise, a derived node, an `IN` node, or an `OUT` node. The method does not filter by truth value.

Edge cases:
- If `node_id` is a premise, returns `[node_id]`.
- If `node_id` has justifications but all antecedent references are missing from `self.nodes`, returns `[]`.
- If a reachable node has a justification with no antecedents, it is not returned as a premise.
- If the graph is cyclic, `visited` prevents infinite recursion.

## Return Value

Returns `list[str]`, a unique list of premise IDs.

Properties:
- Order is depth-first based on iteration over `node.justifications` and each `j.antecedents`.
- Not alphabetically sorted.
- Not a set, despite the docstring wording.
- May be empty if no structurally defined premise is reachable.
- May contain retracted premises.
- Does not include outlist nodes.
- Does not include intermediate derived nodes.

Callers should wrap it in `set(...)` if set semantics are needed, and should filter by `truth_value` or justification validity if they need active support only.

## Algorithm

1. Validate that `node_id` exists in `self.nodes`; raise `KeyError` otherwise.
2. Initialize:
   - `premises = []`
   - `visited = set()`
3. Define recursive helper `_walk(nid)`:
   - If `nid` is already visited or not in `self.nodes`, return.
   - Mark `nid` visited.
   - Look up `node = self.nodes[nid]`.
   - If `not node.justifications`:
     - This is a premise.
     - Append `nid` to `premises` if not already present.
     - Return.
   - Otherwise, for every justification `j` in `node.justifications`:
     - For every `ant_id` in `j.antecedents`:
       - Recurse into `_walk(ant_id)`.
4. Call `_walk(node_id)`.
5. Return `premises`.

Pseudocode:

```python
def trace_assumptions(node_id):
    if node_id not in self.nodes:
        raise KeyError(...)

    premises = []
    visited = set()

    def _walk(nid):
        if nid in visited or nid not in self.nodes:
            return
        visited.add(nid)

        node = self.nodes[nid]

        if not node.justifications:
            if nid not in premises:
                premises.append(nid)
            return

        for j in node.justifications:
            for ant_id in j.antecedents:
                _walk(ant_id)

    _walk(node_id)
    return premises
```

Complexity is `O(V + E)` over reachable nodes and antecedent edges, with recursion depth up to the longest backward chain. There is no memoization across calls.

## Side Effects

None on the network.

- Does not mutate `self.nodes`.
- Does not change truth values.
- Does not modify justifications.
- Does not log.
- Does not call propagation.
- Only local `premises` and `visited` state is created.

The method is read-only apart from potential exceptions.

## Error Handling

- Raises `KeyError` if the requested `node_id` is not in `self.nodes`.
- Silently ignores missing antecedent IDs encountered during `_walk`.
- Does not catch `RecursionError`; very deep graphs can exceed Python’s recursion limit.
- Assumes justification objects are well-formed. If `j.antecedents` is missing or not iterable, Python will raise `AttributeError` or `TypeError`.
- Does not validate that justifications are valid, that antecedents are `IN`, or that outlist nodes are `OUT`.

This asymmetry is important: the starting node is validated, but dangling references inside justifications are tolerated.

## Usage Patterns

Typical direct usage:

```python
assumptions = network.trace_assumptions("claim-123")
```

Primary internal usage is in `find_culprits`:

```python
assumptions = self.trace_assumptions(nid)
assumptions_by_node[nid] = assumptions
all_premises.update(assumptions)
```

There, it is used to map each `IN` nogood node back to its premises, then candidate premises are ranked by entrenchment. Because `trace_assumptions` follows all justifications, `find_culprits` can over-approximate the set of premises responsible for a node. If a node is `IN` through one valid justification but also has invalid or alternative justifications, premises from those alternatives are still returned.

Caller obligations:
- Handle `KeyError` for missing start nodes.
- Do not assume the result is sorted.
- Do not assume returned premises are currently `IN` or retractable.
- Do not assume this traces only the active supporting justification. For that, use `explain` or filter with `_justification_valid`.
- If the caller needs only active support, it should inspect `supporting_justification` or validity-aware traversal instead.

## Dependencies

Internal dependencies:
- `self.nodes: dict[str, Node]`
- `Node.justifications`
- `Justification.antecedents`

The method does not use:
- `_justification_valid`
- `_compute_truth`
- `node.truth_value`
- `Justification.outlist`
- `supporting_justification`

No external modules are involved.

## Topics to Explore

- [function] `reasons/network.py:find_culprits` — Shows how `trace_assumptions` feeds dependency-directed backtracking and candidate ranking.
- [function] `reasons/network.py:_justification_valid` — Explains why tracing all antecedents is broader than tracing only the currently valid support.
- [function] `reasons/network.py:explain` — Contrasts a validity-aware, supporting-justification explanation with this syntactic premise trace.
- [file] `tests/test_backtracking.py` — Exercises contradiction-resolution paths that depend on assumption tracing.
- [general] `truth-maintenance-system-semantics` — Background on SL/CP justifications, outlist defeaters, and premise-based TMS reasoning.

## Beliefs

- `trace-assumptions-requires-start-node-existence` — It raises `KeyError` only for the requested node; missing antecedent IDs encountered during traversal are silently skipped.
- `trace-assumptions-traverses-all-antecedent-edges` — It follows every `Justification.antecedents` entry regardless of justification type, validity, or `supporting_justification`.
- `trace-assumptions-ignores-outlist` — Nodes listed in `Justification.outlist` are never traversed or returned as assumptions.
- `trace-assumptions-defines-premise-structurally` — A premise is a node with an empty `justifications` list, not a node with empty antecedents or `truth_value == "IN"`.
- `trace-assumptions-returns-dfs-list` — It returns a unique list in depth-first discovery order, not a sorted set, despite the docstring saying “set”.
