---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _entrenchment in reasons/network.py

# `Network._entrenchment` — Explanation

## 1. Purpose

`_entrenchment` is a **private scoring heuristic** that ranks how "hard to retract" a node is. It exists to drive two related behaviors implemented in the same `Network` class:

- `find_culprits` computes an `entrenchment` value for each premise that could resolve a contradiction, then **sorts ascending** so the *least*-entrenched premises are listed first.
- `add_nogood` (via `find_culprits`) retracts `culprits[0]` — the least-entrenched premise — when a contradiction is discovered.

The design intent (per the docstring) is a policy: **premises/evidence should be protected; speculative derived nodes should be retracted first**. The function encodes that policy as a single integer, combining several independent signals (premise-ness, sourcing, fan-out, and an explicit `beliefs_type` taxonomy) with hard-coded weights.

It is only called from `find_culprits` in this file — a single-caller helper.

## 2. Contract

- **Preconditions:** None enforced. `node_id` is declared `str`, but any hashable object is accepted (dict membership check). The only real requirement is that `self.nodes` is a `dict[str, Node]`.
- **Postconditions:** Returns a **non-negative `int`**. No mutation of `self.nodes` or the node. No caching — the score always reflects the live state of `node.dependents`, `node.source`, etc.
- **Invariants (implicit):**
  - Any *existing* node scores **at least 20** (from the `type_scores.get(..., 20)` default). A *missing* node scores `0`. So `0 < any existing node's score`, meaning "missing ⇒ least entrenched."
  - Score is monotonic in `len(node.dependents)` (each dependent adds exactly 10).
  - An empty-string `source`/`source_hash` contributes 0 (falsy), not a partial score.

## 3. Parameters

- `node_id: str` — the ID of the node to score.
  - **Edge case (handled):** unknown ID → return `0`.
  - **Edge case (not enforced by types):** non-`str` but hashable (e.g., `None`) passes the `in` check and returns `0`; unhashable (e.g., a `list`) would raise `TypeError` at `node_id not in self.nodes`.

## 4. Return Value

Returns an `int` score built from additive components:

| Component | Condition | Points |
|---|---|---|
| Premise bonus | `not node.justifications` | `+100` |
| Source | `bool(node.source)` is truthy | `+50` |
| Source hash | `bool(node.source_hash)` is truthy | `+25` |
| Fan-out | `len(node.dependents)` | `+10 each` |
| `beliefs_type` | `node.metadata["beliefs_type"].upper()` | `AXIOM/WARNING: 90`, `OBSERVATION: 80`, `DERIVED: 40`, `PREDICTED: 30`, `NOTE: 10`, **any other value / missing: 20** |
| Missing node | `node_id not in self.nodes` | `0` (early return) |

Return `0` only for missing nodes. Any existing node returns `>= 20`.

The caller (`find_culprits`) stores this value in the `"entrenchment"` key of candidate dicts and sorts by it ascending — so **low score = retract first**.

## 5. Algorithm

1. Guard: if `node_id` is not in `self.nodes`, return `0`.
2. Look up `node = self.nodes[node_id]`; initialize `score = 0`.
3. If `node.justifications` is empty (a premise), add `100`. Premises are treated as "evidence" and heavily protected.
4. If `node.source` is truthy, add `50`. If `node.source_hash` is truthy, add another `25`. These are **independent** checks — a node with both gets `+75`.
5. Add `10 * len(node.dependents)`. This term is **unbounded** — a hub node can dominate the whole score.
6. Read `node.metadata.get("beliefs_type", "")`, uppercase it, and add a fixed bonus from the `type_scores` table. Missing key → `""` → not in the table → default `20`.
7. Return the sum.

Note there is no `max()`, clamping, or normalization — the scale is arbitrary but consistent across nodes within one call to `find_culprits`.

## 6. Side Effects

**None.** Pure read of `self.nodes[node_id]` and its attributes (`justifications`, `source`, `source_hash`, `dependents`, `metadata`). No logging (`_log` is not called), no state mutation, no I/O.

## 7. Error Handling

- **No exceptions are raised for its nominal error case** (missing node) — it silently returns `0`.
- **Latent exceptions** that would propagate to the caller:
  - `TypeError` if `node_id` is unhashable (dict membership).
  - `AttributeError` if `node.metadata.get("beliefs_type", "")` returns a non-`str` (e.g., `None` or a list) — `.upper()` would fail. The type system does not enforce that `beliefs_type` is a string.
  - `TypeError` if `node.dependents` is not `Sized` (e.g., `None`). In practice this is a `set` (see `_rebuild_dependents`, `add_node`), but that's an assumption, not enforced here.

## 8. Usage Patterns

- The **only caller** is `find_culprits`, which iterates premises produced by `trace_assumptions(nid)`. Because `trace_assumptions` returns only *nodes with no justifications*, **every node scored via the primary path is a premise** — the `+100` premise branch is *always* true in normal use. The other branches (`source`, `source_hash`, `dependents`, `beliefs_type`) then discriminate *between* premises.
- `find_culprits` sorts with `candidates.sort(key=lambda c: c["entrenchment"])`. Python's sort is **stable**, but candidates originate from iterating the set `all_premises` — **set iteration order is not deterministic across runs/processes**, so equal-entrenchment ties may resolve differently between runs. Do not rely on tie order for reproducible retraction victims.
- **Caller obligation:** the caller must handle `0` (missing node) — but in practice this is a defensive branch that `find_culprits` cannot hit, since `all_premises` is populated only from `self.nodes` lookups in `trace_assumptions`.
- Note a **docstring/implementation mismatch** in `add_nogood`: it claims it "retracts the premise with the fewest dependents," but it actually retracts the **lowest-entrenchment** premise (via `culprits[0]`), and entrenchment includes sourcing, belief-type, and a 100-point premise bonus — not just dependent count.

## 9. Dependencies

- **Internal:** `self.nodes` (the `dict[str, Node]` core data structure); `Node` attributes `justifications`, `source`, `source_hash`, `dependents`, `metadata`.
- **Caller graph:** `find_culprits` → `_entrenchment`; `add_nogood` → `find_culprits`; `trace_assumptions` provides the candidate set.
- **External:** None (no imports used inside the function; only stdlib dict/list behavior).

---

## Assumptions Not Enforced by the Type System

- `node.metadata` is a `dict` and `beliefs_type`, if present, is a **`str`**.
- `node.dependents` is a **`set`** (so `len` counts *unique* dependents; a list with duplicates would inflate the score).
- `node.source` / `node.source_hash` are strings whose truthiness means "present and non-empty" (a whitespace-only string like `" "` counts as present → +50).
- Callers only pass IDs of **existing** nodes in production paths, making the `return 0` branch dead in practice.

---

## Topics to Explore

- [function] `reasons/network.py:find_culprits` — The only caller; how entrenchment drives candidate ordering and what it does with ties.
- [function] `reasons/network.py:trace_assumptions` — Generates the premise set scored here; confirms that only premises are ever scored in the primary path.
- [function] `reasons/network.py:add_nogood` — End-to-end contradiction resolution; exposes the docstring/behavior mismatch about "fewest dependents."
- [file] `reasons/network.py` — The `Node` model and `_compute_truth`/`_propagate` logic that define how retraction cascades.
- [file] `tests/test_backtracking.py` — Expected behavior for culprit selection; verify the weights and ordering against tests.

## Beliefs

- `entrenchment-is-uncached` — `_entrenchment` recomputes from live node state on every call; there is no memoization, so score reflects `dependents`/`source` changes immediately.
- `missing-node-scores-zero` — A nonexistent `node_id` returns `0`, strictly lower than any existing node's minimum score of `20`.
- `premise-bonus-applies-to-all-scored-nodes` — Because `find_culprits` only scores nodes from `trace_assumptions` (premises), the `+100` premise bonus fires for every node scored in the primary path.
- `source-and-source-hash-are-additive` — `source` (+50) and `source_hash` (+25) are independent truthiness checks; a node can receive both.
- `beliefs-type-lookup-is-case-insensitive` — `beliefs_type` is uppercased before the table lookup, so `"axiom"` and `"AXIOM"` both score `90`.
- `entrenchment-ties-are-nondeterministic` — Equal scores sort by insertion order from a `set` in `find_culprits`, so tie-breaking is not reproducible across runs.
