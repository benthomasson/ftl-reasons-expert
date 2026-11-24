---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: assert_node in reasons/network.py

# `assert_node` — Network.assert_node

## Purpose

`assert_node` is the **manual force-in** operation on the dependency network. It sets a node's truth value to `"IN"` regardless of the node's justifications and then lets the standard propagation machinery cascade the change to dependents. It exists as the logical counterpart to `Network.retract`:

- `retract` forces a node `"OUT"` and cascades the resulting invalidations.
- `assert_node` forces a node `"IN"` and cascades the resulting restorations.

It is a user/agent-level override ("un-retract this belief") rather than a re-derivation. If you want truth values recomputed from justifications, you would use `recompute_all` or `add_justification` — not `assert_node`.

## Contract

**Preconditions**
- `node_id` is a key present in `self.nodes`.
- `self.nodes[node_id]` is a `Node` with a `truth_value` attribute that is expected to be one of `"IN"` / `"OUT"` (unenforced by the type system — it is just a `str`).
- `self.nodes[node_id].metadata` is a mutable `dict`.

**Postconditions (when the node was `OUT`)**
- `node.truth_value == "IN"`.
- `"_retracted"` is removed from `node.metadata` (so `_propagate` will no longer skip it as a dependent).
- `node.retracted_at == ""`.
- `node.updated_at` is refreshed to the current UTC time at second resolution.
- An entry is appended to `self.log` with `action="assert"`.
- The node is `IN`, so dependents whose justifications reference it are re-evaluated via `_propagate`; any whose truth value flips are recorded and further cascaded.
- **Note:** `node.metadata["retract_reason"]` (if it was set by `retract`) is **not** cleared.

**Postconditions (when the node was already `IN`)**
- Function returns `[]` immediately with **no side effects**: no mutation, no timestamp refresh, no log entry, no propagation. The operation is effectively idempotent.

**Invariants maintained**
- Every id in the returned `changed` list (except the asserted node itself) is a node whose `truth_value` actually flipped during this call.
- If the change touches dependents whose `_retracted` flag is set, `_propagate` will skip them (they remain `OUT`).

## Parameters

- `node_id: str` — The ID of the node to assert. Must exist in `self.nodes`. Cannot be `None`; no validation of format is performed. Case-sensitive string comparison.

## Return Value

`list[str]` — all node IDs whose truth value changed as a result of this call.

- Empty list iff the node was already `"IN"`.
- Otherwise: the asserted `node_id` is always the **first** element, followed by any dependents that flipped (in BFS order from `_propagate`). Callers should treat the return as the set of "affected" nodes and may use it for audit / display / pub/sub notifications.

Caller obligations:
- Must handle `KeyError` if the ID is untrusted (e.g. from user input).
- Should not assume the asserted node remained `IN` after further `retract` / `challenge` calls elsewhere in the network.

## Algorithm

1. **Exist check.** If `node_id` not in `self.nodes`, raise `KeyError`.
2. **Idempotency short-circuit.** If `node.truth_value == "IN"`, return `[]` without touching state.
3. **Force IN.** Set `node.truth_value = "IN"` (this bypasses `_compute_truth`, so justifications are **not** re-checked).
4. **Clear retraction metadata.** `node.metadata.pop("_retracted", None)` — the flag consumed by `_propagate` to prevent re-evaluating retracted nodes. `node.retracted_at = ""`.
5. **Touch updated_at.** Refresh `node.updated_at` with `datetime.now(timezone.utc).isoformat(timespec="seconds")`.
6. **Seed `changed`.** `changed = [node_id]`.
7. **Log.** `self._log("assert", node_id, "IN")` records to `self.log`.
8. **Propagate.** `changed.extend(self._propagate(node_id))` — BFS over `node.dependents`. For each dependent not already visited and not `_retracted`:
   - Recompute via `_compute_truth(dep)` (which requires all antecedents IN **and** all outlist members OUT).
   - If the value flipped, mutate it, log `"propagate"`, and enqueue that dependent.
   - Skipped dependents (already `_retracted`) are not re-evaluated and stay `OUT` even if their justifications would now be valid — they need their own `assert_node`.
9. **Return** `changed`.

## Side Effects

- Mutates `node.truth_value`, `node.metadata`, `node.retracted_at`, `node.updated_at`.
- Mutates the truth value (+ `supporting_justification` index) of dependent nodes through `_compute_truth`.
- Appends entries to `self.log` (assert plus zero or more `propagate` events).
- Does **not** call `_rebuild_dependents` — the dependents reverse index is assumed up to date.
- No I/O, no network, no filesystem access.

## Error Handling

- **Raises** `KeyError(f"Node '{node_id}' not found")` if the node is missing.
- **Silently no-ops** when the node is already `IN` (returns `[]`). This masks the fact that a retract-reason may be stale — the caller cannot tell from the return value whether anything happened.
- Other errors are not caught: if `node.metadata` is not a `dict`, or `datetime` fails to format, that will bubble up.
- `_propagate` logs a `"warn"` entry (does not raise) for dangling dependents that reference deleted nodes.
- **Asymmetry with `retract`:** `retract` overwrites `node.retract_reason` metadata when a reason is passed. `assert_node` does **not** remove `retract_reason`, so a subsequently asserted node may believe it still has a retract reason. Similarly, `retract` on an already-`OUT` node just refreshes metadata; `assert_node` on an already-`IN` node does nothing at all.

## Usage Patterns

- **User "undo retract":** After `retract(x)`, calling `assert_node(x)` restores `x` and may restore dependents for which `x` was the sole antecedent.
- **Post-import reconciliation:** If a bulk import or `recompute_all` incorrectly dropped a node to `OUT`, `assert_node` restores it.
- **NOT for re-derivation.** Because it bypasses `_compute_truth`, a derived node with all-`OUT` antecedents can still be `"IN"` immediately after `assert_node`. This "forced" state persists until the next `_propagate`, `recompute_all`, `add_justification`, or explicit `retract` re-evaluates it. If the caller needs consistency with justifications, they must call `recompute_all()` afterwards.
- Typically called from the CLI/API where the caller has already validated the node exists, or wrapped in a try/except.
- Dependents that are themselves `_retracted` are never revived by this call — restoring a whole subtree requires iterating `assert_node` over each retracted node (or similar).

## Dependencies

- `datetime`, `timezone` from stdlib (used for `updated_at`).
- `self._log` — appends to `self.log` audit trail.
- `self._propagate` — BFS propagator; the primary cascade engine.
- `self._compute_truth` (indirectly, inside `_propagate`) — checks `_justification_valid` on dependents.
- `Node` fields: `.truth_value`, `.metadata`, `.retracted_at`, `.updated_at`.

## Assumptions Not Enforced by the Type System

- `truth_value` is a `str` but only `"IN"`/`"OUT"` are semantically valid. A typo like `"in"` would never short-circuit and would be treated as `OUT`.
- `metadata["_retracted"]` is a **boolean convention**, not declared anywhere; only its presence matters here.
- `_propagate` returns `list[str]` but is not signaled in the signature; if a subclass overrode it, this contract is implicit.
- The reverse `dependents` index is assumed up to date — `assert_node` will not correct it (unlike `add_node` / `add_justification` / `_rebuild_dependents`).
- Node IDs are assumed unique (guaranteed by `add_node` and `challenge`).
- `datetime` is not monotonic — rapid successive calls can produce identical `updated_at` values, so subscribers should not assume uniqueness of timestamps.

## Topics to Explore

- [function] `reasons/network.py:_propagate` — BFS engine that turns a single truth change into a cascade; key to understanding what `assert_node` actually does downstream.
- [function] `reasons/network.py:retract` — The mirror operation; contrast the metadata it writes (`retract_reason`, `_retracted`) with what `assert_node` clears.
- [function] `reasons/network.py:_compute_truth` — Explains why an asserted derived node can be immediately recomputed back to `OUT` by propagation or `recompute_all`.
- [function] `reasons/network.py:recompute_all` — Shows how a fixpoint re-derivation can undo manual `assert_node` overrides.
- [file] `tests/test_network.py` — Likely contains the assertions-for-`assert_node` idempotency and cascade tests worth reading before changing behavior.

## Beliefs

- `assert-node-raises-keyerror-for-missing-node` — Calling `assert_node("nope")` raises `KeyError(f"Node 'nope' not found")`.
- `assert-node-is-idempotent-when-node-already-in` — If `node.truth_value == "IN"`, the function returns `[]` with no logged event and no timestamp update.
- `assert-node-does-not-clear-retract-reason` — `metadata["_retracted"]` is popped and `retracted_at` cleared, but `metadata["retract_reason"]`, if previously set by `retract(..., reason=...)`, is left in place.
- `assert-node-does-not-invoke-compute-truth-on-target` — The asserted node's `truth_value` is set directly to `"IN"`, bypassing justification validity checks; only dependents are re-evaluated.
- `assert-node-returns-asserted-node-first` — The returned `changed` list always begins with `node_id` when it is non-empty, followed by BFS-ordered dependent flips.
- `assert-node-skips-retracted-dependents` — `_propagate` continues past dependents whose `metadata["_retracted"]` is truthy, so they remain `OUT` after an assert cascade.
