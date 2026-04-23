# Function: _propagate in reasons_lib/network.py

**Date:** 2026-04-23
**Time:** 16:34

## `_propagate` — BFS Truth Value Propagation

### 1. Purpose

`_propagate` is the engine that maintains consistency in the dependency network after a node's truth value changes. When a node goes IN→OUT (retraction) or OUT→IN (restoration), any node that *depends* on it may need to change too. This method walks the dependency graph forward (from cause to effect) and recomputes each dependent's truth value, cascading further if that recomputation itself causes a change.

It implements the forward-chaining propagation step of Doyle's Truth Maintenance System (TMS). Without it, retracting a premise would leave downstream conclusions stale — they'd still say "IN" even though their support vanished.

### 2. Contract

**Preconditions:**
- `changed_id` must be a key in `self.nodes` (no guard — will raise `KeyError` if missing).
- The node at `changed_id` must have *already* had its truth value updated before `_propagate` is called. This method does not change the trigger node — only its transitive dependents.

**Postconditions:**
- Every reachable dependent whose truth value *would* change (given current justifications) has been updated in place.
- The returned list contains exactly the IDs of nodes that changed, in BFS discovery order.
- No retracted nodes (`_retracted` in metadata) are recomputed — they stay OUT regardless of what their justifications say.

**Invariant:** After `_propagate` returns, for every non-retracted derived node reachable from `changed_id`, `node.truth_value == self._compute_truth(node)`.

### 3. Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `changed_id` | `str` | The node whose truth value just changed. Used as the BFS seed. Its own truth value is *not* recomputed — only its dependents are. |

**Edge cases:**
- If the node has no dependents, the method returns `[]` immediately (the `while` loop body never enters the `for`).
- If `changed_id` refers to a node with dependents that are all retracted, it returns `[]`.

### 4. Return Value

`list[str]` — IDs of all nodes whose truth value changed during propagation, in BFS order. The trigger node (`changed_id`) is *never* in this list. Callers (like `retract` and `assert_node`) prepend `changed_id` to their own `changed` list separately, then extend with this result.

### 5. Algorithm

1. **Initialize BFS state.** A `deque` queue seeded with `changed_id`, and a `visited` set (also containing `changed_id`) to prevent cycles and redundant work.

2. **Dequeue the current node.** Look up its `dependents` — the set of node IDs that have this node in one of their justifications' antecedent or outlist.

3. **For each dependent:**
   - **Skip if visited.** Prevents infinite loops in cyclic dependency graphs and avoids recomputing a node that was already handled.
   - **Skip if retracted.** Nodes with `_retracted` metadata are manually held OUT. Their truth value is not recomputed — they stay OUT until explicitly re-asserted via `assert_node`.
   - **Recompute truth value** via `_compute_truth`, which checks whether *any* justification is currently valid.
   - **If changed:** update the node's `truth_value` in place, record it in `changed`, log it, mark visited, and enqueue it so *its* dependents are checked next.
   - **If unchanged:** do nothing — the cascade stops along this path.

4. **Return** the accumulated list of changed node IDs.

Key insight: the cascade is *selective*. It only continues through nodes that actually change. If node A depends on B and C, and B goes OUT but C's justification independently keeps A IN, the cascade stops at A.

### 6. Side Effects

- **Mutates `dep.truth_value`** in place for every node that changes.
- **Appends to `self.log`** via `_log("propagate", ...)` for each changed node — this is the audit trail.
- No I/O, no disk writes, no external calls.

### 7. Error Handling

None. The method assumes:
- `changed_id` exists in `self.nodes` (callers guarantee this).
- Every ID in `current.dependents` exists in `self.nodes` (maintained by `add_node`, which only adds to `dependents` for nodes already in the network).

If either assumption is violated, you get an unguarded `KeyError`. This is intentional — a broken invariant in the dependency graph is a bug, not a recoverable error.

### 8. Usage Patterns

Every method that changes a node's truth value calls `_propagate` afterward:

| Caller | Trigger |
|--------|---------|
| `retract()` | Node set to OUT, propagate cascade |
| `assert_node()` | Node set to IN, propagate restoration |
| `supersede()` | Old node may go OUT via outlist |
| `add_justification()` | Node may change truth value with new support |
| `challenge()` | Target may go OUT due to challenge in outlist |
| `convert_to_premise()` | Node may come back IN |

The pattern is always:
```python
node.truth_value = new_value   # update the trigger
changed = [node_id]
changed.extend(self._propagate(node_id))
```

The caller is responsible for updating the trigger node *before* calling `_propagate`. The method only handles downstream effects.

### 9. Dependencies

- **`collections.deque`** — used for O(1) BFS queue operations.
- **`self._compute_truth(node)`** — evaluates whether any justification for a node is currently valid. This delegates to `_justification_valid`, which checks antecedent IN-ness and outlist OUT-ness.
- **`self._log()`** — audit logging, appends to `self.log`.
- **`Node.dependents`** — a `set[str]` maintained by `add_node` (and `supersede`, `add_justification`, `challenge`) that tracks the reverse dependency edges.

---

## Topics to Explore

- [function] `reasons_lib/network.py:_compute_truth` — The truth evaluation function that `_propagate` calls at each step; understanding its semantics (especially the outlist / non-monotonic reasoning) is essential to predicting cascade behavior
- [function] `reasons_lib/network.py:recompute_all` — An alternative to BFS propagation that iterates to a fixpoint over all nodes; compare when each strategy is appropriate
- [function] `reasons_lib/network.py:retract` — The primary caller that sets `_retracted` metadata before propagating; understanding the retract/assert lifecycle clarifies why `_propagate` skips retracted nodes
- [file] `tests/test_network.py` — Test cases that exercise cascade and restoration scenarios, showing concrete examples of multi-hop propagation
- [general] `doyle-tms-algorithm` — The 1979 paper by Jon Doyle on Truth Maintenance Systems that this implementation follows; understanding SL justifications and dependency-directed backtracking gives the theoretical foundation

## Beliefs

- `propagate-skips-retracted-nodes` — `_propagate` never recomputes truth values for nodes with `_retracted` in metadata, even if their justifications would make them IN; only `assert_node` can bring them back
- `propagate-does-not-change-trigger` — The seed node (`changed_id`) is added to `visited` immediately and never has its own truth value recomputed; callers must update it before calling `_propagate`
- `propagate-cascade-stops-on-unchanged` — If a dependent's recomputed truth value equals its current value, it is not enqueued; the cascade terminates along that path
- `propagate-bfs-order-guarantee` — Changed nodes are returned in BFS discovery order, meaning closer dependents appear before transitive ones
- `propagate-assumes-dependents-exist` — Every ID in `node.dependents` is accessed via `self.nodes[dep_id]` without a membership check; a dangling dependent reference will raise `KeyError`

