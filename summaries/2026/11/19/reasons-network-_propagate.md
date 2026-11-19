---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _propagate in reasons/network.py

# `_propagate` — BFS Truth Value Propagation

## Purpose

`_propagate` is the engine of the Truth Maintenance System (TMS). When a node's truth value (`IN`/`OUT`) changes — because it was asserted, retracted, superseded, given a new justification, etc. — any node whose justifications reference it may now have a different truth value itself. `_propagate` walks **forward through the reverse-dependency index** (`node.dependents`) in BFS order, recomputing each reachable dependent's truth value and cascading the change further whenever a dependent's value actually changes.

It is the single mechanism that implements Doyle's 1979 "retraction cascade" and "restoration cascade" for the network.

## Contract

**Preconditions**
- `changed_id` must exist in `self.nodes` (this function dereferences `self.nodes[changed_id]` without a guard).
- The caller must have *already mutated* `changed_id`'s truth value (or its justifications) before calling — `_propagate` does **not** recompute `changed_id` itself and does **not** include it in its return list. That responsibility is on the caller (see `retract`, which does `changed = [node_id]` then extends with the propagation result).
- The reverse-index (`node.dependents`) should be consistent with `justifications`. If it is stale, propagation is incomplete or over-broad. `_rebuild_dependents`/`verify_dependents` exist to diagnose this.
- `_propagate` must not be re-entered while the graph is half-mutated (single-threaded, callables from `retract`/`assert`/`supersede`/etc. satisfy this).

**Postconditions**
- Every node reachable from `changed_id` via the `dependents` relation has had `_compute_truth(dep)` called on it, **unless** it was already `visited` from a value change, was flagged `_retracted`, or was a dangling ID.
- `dep.truth_value` is consistent with the current graph, for every processed node.
- `dep.supporting_justification` may have been mutated by `_compute_truth`.
- `self.log` gains one `"propagate"` entry per changed node and one `"warn"` entry per dangling dependent.
- The returned list contains the IDs whose value actually changed. It does **not** include `changed_id`.

**Invariants maintained**
- BFS terminates in at most `N` enqueues because `visited` grows monotonically and only nodes whose value *changed* are added to the queue.
- Idempotent given identical graph state: re-running with a fully settled graph returns `[]`.
- `changed_id` is protected against self-loop via initial `visited = {changed_id}`.

## Parameters

- **`changed_id: str`** — ID of the node whose truth value or justifications the caller just modified. Must be a key in `self.nodes`. Edge case: if the changed node's ID stub is missing from `self.nodes`, a `KeyError` propagates (not caught here). Edge case: if `changed_id` is itself retracted (`_retracted == True`), it is still processed as `current` (the `_retracted` check only applies to *dependents*, not to the seed) — this is how `retract()` reuses it to update dependents of a freshly-retracted node.

## Return Value

A `list[str]` of node IDs whose `truth_value` was changed during the BFS, **excluding** `changed_id`. Order is non-deterministic (depends on `set` iteration order over `current.dependents`) and reflects the order in which BFS visited each changed node. Duplicates are impossible (each ID is added at most once, guarded by `visited`). Callers typically concatenate: `changed = [node_id]; changed.extend(self._propagate(node_id))`.

## Algorithm

1. Initialize `changed = []`, `queue = deque([changed_id])`, `visited = {changed_id}`.
2. Pop `current_id` from the left of the queue (BFS).
3. For each `dep_id` in `self.nodes[current_id].dependents`:
   - If `dep_id in visited` → skip (already processed, or a cycle).
   - If `dep_id not in self.nodes` → log `"warn"` with `"dangling dependent of {current_id}"` and skip (does **not** add to `visited`, so it can re-warn for each parent).
   - If `dep.metadata.get("_retracted")` → skip silently (retracted nodes stay OUT; they are neither visited nor enqueued).
   - Otherwise compute `old_value = dep.truth_value` and `new_value = self._compute_truth(dep)`.
   - If they differ: assign `dep.truth_value = new_value`, append `dep_id` to `changed`, log `"propagate"`, add `dep_id` to `visited`, enqueue it.
4. Repeat until the queue drains, then return `changed`.

## Side Effects

- Mutates `dep.truth_value` for each changed dependent.
- Mutates `dep.supporting_justification` via `_compute_truth` (it sets `node.supporting_justification` as a side effect even if the value ends up unchanged — a subtle, undocumented mutation path).
- Appends dicts to `self.log` (`"propagate"` per change, `"warn"` per dangling dependent, with ISO timestamps).
- No I/O, no external calls, no exception raising given valid preconditions.

## Error Handling

- **`KeyError`** if `changed_id` is not in `self.nodes` (uncaught; caller responsibility).
- **`KeyError`** could also occur if a node on the queue is removed between iterations — not possible in synchronous single-threaded use, but no defensive guard exists.
- **No return-based errors.** Every anomalous condition is either swallowed (`_retracted`, `visited`) or logged-and-skipped (dangling dependent). A dangling dependent does **not** abort propagation of other dependents.
- **Termination is guaranteed** by the `visited` set; cycles in the justification graph (A ← B ← A) simply terminate when the cycle closes back on a visited node.

## Usage Patterns

Called exclusively from public mutators on `Network`, in each case *after* mutating the affected node:

- `retract(node_id)` — sets `_retracted` + `OUT`, then propagates.
- `assert_node(node_id)` — clears `_retracted` + sets `IN`, then propagates.
- `supersede`, `add_justification`, `challenge`, `convert_to_premise`, `remove_justification` — same pattern: mutate the target's value/justifications in place first, then call `_propagate`.

Callers must therefore (a) not double-count `changed_id` (they prepend it), and (b) ensure the reverse index `dependents` is updated *before* calling `_propagate`, otherwise propagation misses or misroutes dependents.

## Assumptions Not Enforced by the Type System

1. `node.dependents` is a `set[str]` of IDs that are (usually) keys in `self.nodes` — the "usually" is exactly what the dangling check guards; nothing prevents typos from creeping in.
2. `node.metadata["_retracted"]` is a convention, not a field. A retracted node without that metadata key would participate in propagation.
3. `_compute_truth(dep)` is assumed to be a pure re-derivation from the current graph — the function does not detect whether the caller forgot to update some antecedent.
4. Callers are assumed to hold no lock/invariant violation across the call; nested `_propagate` calls would work but confuse the `changed` bookkeeping.
5. `changed_id` was already updated by the caller — this is a *soft* contract.

## Dependencies

- `collections.deque` — BFS queue.
- `self.nodes: dict[str, Node]` — authoritative node registry.
- `Node.dependents`, `Node.truth_value`, `Node.metadata`, `Node.supporting_justification` — the node model.
- `self._compute_truth` → `self._justification_valid` — the truth evaluation kernel.
- `self._log` — audit-trail writer (writes `datetime.now()`).

## Topics to Explore

- [function] `reasons/network.py:_compute_truth` — The kernel that decides a node's IN/OUT from its justifications; note the side effect on `supporting_justification` that `_propagate` inherits.
- [function] `reasons/network.py:_rebuild_dependents` — The canonical reverse-index builder; if `_propagate` behaves oddly, the index is the first suspect.
- [function] `reasons/network.py:recompute_all` — An alternative "fixpoint over the whole graph" reconciliation path; compare to forward BFS for stamping bootstraps vs incremental edits.
- [file] `tests/test_dangling_dependents.py` — Presumably exercises the `"warn"`-and-skip path when `dependents` reference removed nodes.
- [general] `retracted-node-semantics` — How `_retracted` short-circuits propagation, why retracted nodes are skipped entirely as dependents, and what happens if `_retracted` is set on a node that is later referenced.

## Beliefs

- `propagate-excludes-seed-node` — The seed `changed_id` is never included in `_propagate`'s returned list; callers must prepend it themselves.
- `propagate-skips-retracted-nodes` — Nodes with `metadata["_retracted"] == True` are not recomputed and not enqueued, so their dependents are not visited through them.
- `propagate-requires-reverse-index` — Correct propagation depends on `node.dependents` being up-to-date with justifications; `_rebuild_dependents`/`verify_dependents` exist because this can drift.
- `propagate-guards-cycles-with-visited-set` — The `visited` set (seeded with `changed_id`) is the only safeguard against cycles in the justification graph; termination relies on its monotonic growth.
- `propagate-has-nondeterministic-order` — Because `dependents` is a set, both the BFS traversal order and the returned `changed` list order are non-deterministic across runs/processes.
