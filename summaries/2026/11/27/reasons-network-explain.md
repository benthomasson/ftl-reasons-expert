---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: explain in reasons/network.py

# `Network.explain` — Explanation Tracer

## Purpose

`explain` answers the question *"why is this node IN or OUT?"* by walking the justification graph and producing a **flat list of explanation dicts** describing each step of the reasoning chain. It's the introspection/debugging counterpart to the propagation machinery (`_compute_truth` / `_propagate`): where `_compute_truth` silently sets a truth value, `explain` reconstructs the *narrative* a human can read.

It handles two very different modes:

- **IN node** → finds *one* valid justification, emits a "justification valid" step, and recursively explains each antecedent (drilling toward premises).
- **OUT node** → emits one step *per justification* showing which antecedents failed and which outlist entries were violated. It does **not** recurse in this branch.

## Contract

**Preconditions**
- `node_id` must exist in `self.nodes` (enforced by an explicit `KeyError`).
- Implicitly: the node's `truth_value` must be `"IN"` or `"OUT"` — no other values are handled.
- Implicitly: for an IN node with justifications, at least one justification **must** be currently valid. This is **not** checked — see Assumptions below.

**Postconditions**
- Returns a list of dicts, each with at least `node`, `truth_value`, and `reason`.
- Every node reachable via *valid* IN-justification antecedents is visited at most once (deduplication via `_visited`).
- No mutation of network state (`self.nodes`, `self.log`, etc.). Only the internal `_visited` set grows.

**Invariants preserved**
- Recursion terminates even on cyclic graphs (`_path` guard).
- Sibling branches see independent `_path` (because `_path = _path | {node_id}` creates a **new set** rather than mutating), so a diamond dependency A→B, A→C, B→D, C→D explores D once via `_visited` dedup rather than falsely reporting a cycle.

## Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `node_id` | `str` | ID of the node to explain. Must exist in `self.nodes`; otherwise `KeyError`. |
| `_visited` | `set \| None` | Internal. IDs already explained in this traversal; used to avoid duplicates in DAG-shaped graphs. Caller should leave `None`. |
| `_path` | `set \| None` | Internal. IDs on the *current* recursion path; used to detect cycles. Caller should leave `None`. |

**Edge cases**
- `_visited is None` / `_path is None` are defaulted to fresh empty sets, so external callers use `network.explain(id)`.
- Passing a pre-populated `_visited` would silently suppress explanations for those IDs (this is how internal recursion works, but is not intended for external use).
- Dangling antecedent IDs (referenced but not in `self.nodes`): for the **IN** path, `_justification_valid` rejects justifications that reference missing nodes (it requires `a in self.nodes`), so those justifications are never selected — no crash. For the **OUT** path, `failed` is filtered by `a in self.nodes`, so dangling antecedents are silently excluded from the report.

## Return Value

A `list[dict]`. Possible shapes:

**Premise (no justifications)**
```python
{"node": id, "truth_value": "IN"|"OUT",
 "reason": "premise" | "retracted premise"}
```

**IN node with valid justification**
```python
{"node": id, "truth_value": "IN",
 "reason": "<type> justification valid",   # e.g. "SL justification valid"
 "antecedents": [...], "label": ...,
 "outlist": [...]   # present only if non-empty
}
```

**OUT node, one step per justification**
```python
{"node": id, "truth_value": "OUT",
 "reason": "<type> justification invalid",
 "failed_antecedents": [ids whose value is OUT],
 "label": ...,
 "violated_outlist": [ids whose value is IN]   # present only if non-empty
}
```

**Cycle**
```python
{"node": id, "truth_value": <current>,
 "reason": "circular dependency"}
```

**Empty list** — returned when a node has already been visited on another branch (`node_id in _visited` and not on the current `_path`). Also returned by the IN branch when *no* valid justification is found (see below).

The caller must be prepared for an empty list and for the sequencing behavior where a premise's step appears *after* the derived node step that mentions it (depth-first order, antecedents appended in `j.antecedents` order).

## Algorithm — Step by Step

1. **Existence check** — `KeyError` if `node_id not in self.nodes`. This happens *before* any recursion guards.
2. **Default the guard sets** if `None`.
3. **Cycle check** — if `node_id in _path`, return one `"circular dependency"` step (with the node's *current* truth value).
4. **Dedup check** — if `node_id in _visited`, return `[]`. (Note: cycle check precedes dedup, so a node currently on the path is reported as a cycle rather than skipped.)
5. **Mark visited** — `_visited.add(node_id)`.
6. **Extend path** — `_path = _path | {node_id}` (new set, so sibling branches are isolated).
7. **Premise short-circuit** — if `not node.justifications`, emit a single premise step and return immediately.
8. **IN branch**:
   - Try `node.supporting_justification` if in-range and `_justification_valid(candidate)`.
   - Otherwise scan `reversed(node.justifications)` (newest-first, matching `_compute_truth`) for the first valid one.
   - If a valid `j` is found: emit the "valid" step (recording `antecedents`, `label`, and `outlist` when non-empty), then recurse on **each antecedent** with the same `_visited`/`_path`.
   - If **none** is valid: emit nothing for this node. (This is the silent inconsistency case — see Assumptions.)
9. **OUT branch**:
   - For each justification, compute `failed = [a for a in j.antecedents if a in nodes and <a is OUT>]` and `violated_outlist = [o for o in j.outlist if o in nodes and <o is IN>]`.
   - Emit one step per justification. **No recursion.**
10. Return `steps`.

## Side Effects

- **Mutates `_visited`** — the caller's set (or the fresh one) accumulates every visited node. This is the deduplication mechanism and is observable across sibling branches.
- **No mutation of `_path`** — the reassignment `_path = _path | {node_id}` produces a new set local to this frame.
- **No network state changes** — does not touch `self.log`, truth values, dependents, etc. It's a pure read over `self.nodes` *and* `self._justification_valid` (which itself only reads).
- No I/O, no external calls, no randomness.

## Error Handling

- **`KeyError(f"Node '{node_id}' not found")`** — raised if the entry `node_id` is missing. In recursive calls, antecedents are always filtered by `_justification_valid` (IN) or by `a in self.nodes` (OUT), so missing-node `KeyError`s should not occur internally in a well-formed graph.
- **Silent failure inside IN branch** — if `node.truth_value == "IN"` but no justification is valid, no step is emitted and no antecedent is visited. This is not signalled; it's a symptom of an inconsistent network (truth value disagreeing with justification state).
- **Swallowed dangling references in OUT branch** — antecedents/outlist entries not in `self.nodes` are omitted from `failed_antecedents`/`violated_outlist` rather than reported.

## Usage Patterns

Typical use is introspection / debugging:

```python
steps = network.explain("claim-42")
for step in steps:
    print(step["node"], step["truth_value"], step["reason"])
```

Callers should:
- Not rely on ordering being purely top-down — it's DFS pre-order (parent step first, then antecedents).
- Not rely on cycle detection producing a stable "canonical" cycle representation; it reports the node where the cycle closes.
- Treat empty lists as normal (already-explained or inconsistent IN).

## Dependencies

- `self.nodes` — the node registry.
- `Node` fields: `truth_value`, `justifications`, `supporting_justification`.
- `Justification` fields: `type`, `antecedents`, `outlist`, `label`.
- `self._justification_valid(j)` — the authoritative validity predicate (SL/CP semantics).
- Standard Python `set` operations for guards.

## Assumptions Not Enforced by the Type System

1. **`truth_value ∈ {"IN", "OUT"}`** — anything else falls into the `else` (OUT) branch, producing a misleading "invalid justification" report.
2. **`supporting_justification` is a valid index or `None`** — out-of-range `si` is defensively skipped (`0 <= si < len(...)`), but a valid-looking index pointing at an invalid justification simply falls through to the reversed scan.
3. **An IN node always has ≥1 valid justification** — if not, the IN branch silently emits no step. Callers cannot distinguish "explained nothing because inconsistent" from "explained nothing because already visited" in isolation.
4. **Recursive antecedents always exist in `self.nodes`** — for the IN path this is guaranteed transitively by `_justification_valid` requiring `a in self.nodes`; for OUT no recursion occurs, so the assumption only applies to the IN path.
5. **`_visited` and `_path` are caller-private** — nothing prevents external callers from supplying malformed sets (e.g., a `_path` containing unrelated IDs), which would produce spurious "circular dependency" reports.

## Topics to Explore

- [file] `reasons/network.py` — Full `Network` class; `explain` is one of several introspection methods (`trace_assumptions`, `trace_access_tags`, `find_culprits`) with different recursion policies.
- [function] `reasons/network.py:_justification_valid` — The SL/CP validity semantics that `explain` (and every propagation path) depends on; understands outlist non-monotonicity.
- [function] `reasons/network.py:_compute_truth` — How truth values are assigned and why `supporting_justification` is set; `explain` mirrors its newest-first preference.
- [general] `truth-maintenance-explanation` — How other TMS implementations structure explanations (`support-list` vs `support-graph`) and whether a single-valid-justification IN report loses information.
- [file] `tests/test_outlist.py` — Demonstrates outlist violations that `explain` surfaces in the OUT branch's `violated_outlist` field.

## Beliefs

- `explain-only-recurses-on-in-justifications` — The function drills into antecedents only when the node is IN and a valid justification is found; OUT-mode explanations are one level deep and never recurse.
- `visited-dedupes-across-branches-while-path-detects-cycles` — `_visited` is mutated and shared across all recursive branches, while `_path` is replaced with a new set per frame, giving deduplication and per-branch cycle detection respectively.
- `cycle-check-precedes-visited-check` — A node appearing in `_path` is reported as `"circular dependency"` even if it is also in `_visited`, because the `_path` guard is evaluated first.
- `explain-silently-omits-inconsistent-in-nodes` — If a node's `truth_value` is `"IN"` but `_justification_valid` returns false for every justification, `explain` emits no step and no recursive expansion rather than raising.
- `explain-raises-keyerror-before-recursion-guards` — The `node_id not in self.nodes` check runs before `_path`/`_visited` handling, so any recursive call on a missing antecedent would raise rather than be suppressed (safe only because callers pre-filter antecedents).
