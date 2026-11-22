---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: recompute_all in reasons/network.py

# `Network.recompute_all` — Deep Dive

## Purpose

`recompute_all` is the **global fixpoint sweep** for the TMS (Truth Maintenance System) network. It walks every node that has justifications (i.e., every *derived* node) and asks the justification engine "what should your truth value be right now?" repeatedly, until nothing changes.

It exists because the incremental propagation path (`_propagate`, invoked by `retract`, `assert_node`, `add_justification`, etc.) only BFS-es forward from the node that was just modified. That is fine for a single event, but breaks down when:

- Nodes are loaded in an arbitrary order (e.g., deserialization, import, batch edits),
- A justification was added/removed without linking an antecedent through `dependents`,
- The `dependents` reverse index is stale or was rebuilt after-the-fact.

So `recompute_all` is the **safety net**: a caller-agnostic way to re-derive the entire truth state from scratch, driven only by the justification graph itself rather than by the `dependents` index.

---

## Contract

**Preconditions**
- `self.nodes` is fully populated (the function iterates over it directly).
- `self._compute_truth` is available (it is a method on the same object).
- Caller has not already left the network in an inconsistent state that has no fixpoint (e.g., a justification cycle). No such state is *prevented* by types.

**Postconditions**
- Every node `n` where `n.justifications` is non-empty and `n.metadata["_retracted"]` is falsy has `n.truth_value == network._compute_truth(n)` **as of the last pass** — assuming convergence was reached within `len(self.nodes) + 1` passes.
- `node.supporting_justification` has been updated to the last-valid justification index (side effect of `_compute_truth`).
- `self.log` has one `"recompute"` entry per (node, pass) transition where the value changed.
- Premises (`justifications == []`) are unchanged by this method. Their value (IN/OUT) is preserved verbatim (see `_compute_truth` early-return).
- Nodes with `metadata["_retracted"] == True` are skipped — they keep their current value.

**Invariants preserved**
- A retracted node's truth value is *not* re-derived. This prevents a retracted node from silently flipping back IN due to the justification graph. Restoration is the exclusive domain of `assert_node`.
- Premises never leave `"IN"` (unless previously retracted), because the function never touches premises.

**Invariants *not* enforced**
- Convergence is not verified. If the loop exhausts `max_iterations` without a stable pass, there is no error, warning, or log entry — the caller just gets a possibly-incomplete result.
- Cycle detection is not performed. Justification cycles (A depends on B, B depends on not B, etc.) can oscillate or produce arbitrary results depending on insertion order.
- The `dependents` reverse index is **not** consulted. This method does not care whether `dependents` is up-to-date. This is intentional (that is the point of the "full sweep" mode), but is worth knowing.

---

## Parameters

None (only `self`). `recompute_all` takes no arguments, so it cannot be scoped to a subset of the graph.

Edge cases:
- `self.nodes == {}` → `max_iterations = 1`, first pass iterates over nothing, `changed_this_pass` stays empty, loop breaks. Returns `[]`.
- All nodes are premises → same as above, returns `[]`.
- All nodes are retracted → same, returns `[]`.

---

## Return Value

`list[str]` — the **union over all passes** of node IDs whose `truth_value` differed from `_compute_truth(node)` at the moment that node was visited in that pass.

Important subtleties:

1. **It is a set converted to a list.** Order is a CPython-implementation artifact (set iteration order), not a contract. Do not rely on it.
2. **A node may be reported even if its final value equals its initial value.** If pass 1 changes `A: OUT → IN` and pass 3 changes `A: IN → OUT`, only `"A"` is in `all_changed` — you cannot reconstruct the sequence of intermediate values from the return value. Use `self.log` for that.
3. **The returned list may be incomplete if convergence was not reached.** The function cannot distinguish "converged" from "hit max_iterations." Callers that need certainty should verify by inspecting `_compute_truth` on a fresh pass, or by checking whether the last iteration count equals `max_iterations`.
4. **Nodes that were already correct at start are never returned**, even if `_compute_truth` also rewrote their `supporting_justification` pointer to a different (still-valid) justification.

---

## Algorithm

```
all_changed ← ∅
max_iterations ← |self.nodes| + 1

repeat up to max_iterations times:
    changed_this_pass ← []
    for each (nid, node) in self.nodes (insertion order):
        if node.justifications is non-empty AND node is not retracted:
            old ← node.truth_value
            new ← self._compute_truth(node)          # side effect: sets supporting_justification
            if old ≠ new:
                node.truth_value ← new
                changed_this_pass.append(nid)
                self._log("recompute", nid, new)
    if changed_this_pass is empty:
        break                                        # fixpoint reached
    all_changed ← all_changed ∪ changed_this_pass

return list(all_changed)
```

**Why `len(self.nodes) + 1`?** In a DAG, a cascading change from an updated node to a leaf can traverse at most `N − 1` edges, so an in-order-independent propagation needs at most `N` passes to stabilize. `+1` gives one slack pass. This bound is adequate under the assumption that no justification chain cycles. **That assumption is not checked and not enforced anywhere in the codebase.** Cyclic or non-monotonic justifications (SL justifications with each other in outlists, for instance) can require more passes or oscillate, and the function will simply marshal the state after `N+1` iterations.

**Why is the pass boundary detected by "no changes"?** Because justifications can be satisfied by different nodes across passes — a change to node B on pass *k* may unlock a change to node A that was previously OUT, which then changes B on pass *k+1*. Detecting the fixpoint by a fully clean pass is the simplest correct termination condition for monotone TMS semantics.

**Why iterate over `self.nodes.items()` and not over `dependents`?** Because this is the "full recompute" path, explicitly decoupled from the reverse index. Order does not matter for correctness because the outer loop guarantees eventual propagation in the DAG case.

---

## Side Effects

- **Mutates** `node.truth_value` on every node whose value changes.
- **Mutates** `node.supporting_justification` on every derived node visited, because `_compute_truth` sets it on both the IN and OUT paths (it is set to `None` when no justification is valid). This is not visible in the return value.
- **Appends** to `self.log` one entry per actual transition: `{"timestamp": ..., "action": "recompute", "target": nid, "value": new}`.
- Does **not** call `_propagate` or `_propagate_access_tags`, so `access_tags` are not re-propagated. If you've changed source nodes and expect access tags to cascade, run the tag propagation separately (or call `_propagate_access_tags` on each changed ID).
- Does **not** touch `self.nogoods`, `self.repos`, or `self.meta`.

---

## Error Handling

The function itself raises **no exceptions**. Its exceptions would have to come from within `_compute_truth` / `_justification_valid`. Looking at those:

- `_justification_valid` is defensive: it treats missing antecedent nodes as "not IN" and missing outlist nodes as "OUT" (`o not in self.nodes or …`), so a dangling reference is not an error — it is silently treated as unsatisfied/precondition-met.
- `_compute_truth` can only raise if `node.justifications` is malformed (e.g., an object without `.type`), which is a data-model violation, not something this function defends against.

So error semantics are "best effort, no raises, silently incomplete on non-convergence." That is a design choice worth questioning: a caller cannot tell the difference between a clean fixpoint and a truncated result.

---

## Usage Patterns

Typical callers:

- **After bulk import / deserialization.** Build the node dict, rebuild `dependents`, then `recompute_all()` to normalize truth values in one sweep instead of relying on the pass-order of `add_node`.
- **After manual graph surgery** (`remove_justification`, `convert_to_premise`, direct mutation of `justifications`) when you are not sure incremental propagation reached every dependent.
- **Diagnostics / repair.** After `verify_dependents()` reports missing entries, call this to bring truth values back in line without repairing the reverse index first — the two are orthogonal under this method.

Caller obligations:

- Treat the return value as **informational**. It is the union of transition events, not the final set of truth values. Read `node.truth_value` for the answer.
- If deterministic ordering matters, sort the result (`sorted(recompute_all())`).
- If you need to know whether the fixpoint was reached, count how many `"recompute"` entries were logged before and after and compare passes — or verify with a follow-up manual pass over `self.nodes` calling `_compute_truth`.

---

## Dependencies

- `self.nodes` — canonical node registry; insertion order determines the sweep order.
- `Node.justifications` and `Node.metadata["_retracted"]` and `Node.truth_value` — the state being read/updated.
- `Network._compute_truth` — the truth oracle; also sets `supporting_justification`.
- `Network._justification_valid` — called transitively by `_compute_truth`; defines SL/CP semantics (all antecedents IN and all outlist nodes OUT).
- `Network._log` — appends to the audit trail.
- `Node.supporting_justification` — internally rewritten, not exposed via the return value.

No external modules or I/O. No concurrency guarantees — the mutation of `truth_value` during iteration uses `self.nodes.items()`, which in CPython is safe as long as the set of keys is not modified during iteration (which it isn't here).

---

## Topics to Explore

- [function] `reasons/network.py:_compute_truth` — The oracle this loop relies on; explaining why it mutates `supporting_justification` as a side effect is important context for callers.
- [function] `reasons/network.py:_propagate` — The *incremental* counterpart to `recompute_all`; contrasting the two clarifies when to call which.
- [file] `reasons/network.py` — The entire file: `retract`, `assert_node`, `add_justification`, and `convert_to_premise` all feed into or out of this function's state model.
- [general] `tms-fixpoint-convergence` — Investigate Doyle's original algorithm and the assumption that `N+1` passes suffice; test what happens with justification cycles and retracted-node chains.
- [general] `dependents-index-vs-full-recompute` — Decide when the incremental `dependents`-driven path is trustworthy vs. when the full sweep is required; the `verify_dependents` helper is relevant here.

## Beliefs

- `recompute-all-skips-premises` — Nodes with an empty `justifications` list are never visited or mutated by `recompute_all`; their truth value is preserved.
- `recompute-all-skips-retracted-nodes` — Nodes with `metadata["_retracted"]` truthy are excluded from the sweep, so retracted nodes cannot be revived by this function.
- `recompute-all-ignores-dependents-index` — Unlike `_propagate`, `recompute_all` does not consult `node.dependents`; correctness depends only on `justifications` and `_compute_truth`.
- `recompute-all-mutates-supporting-justification` — Every visited derived node has its `supporting_justification` field rewritten, even when `truth_value` is unchanged, because `_compute_truth` sets it as a side effect.
- `recompute-all-max-iterations-is-a-hard-cap` — The loop stops after `len(self.nodes) + 1` passes and does not raise, warn, or log a signal when convergence is not achieved.
- `recompute-all-return-is-event-union-not-final-state` — The returned list is the union of nodes that ever flipped during the sweep; it can include nodes whose final value equals their initial value and cannot be used as a replacement for reading `node.truth_value`.
