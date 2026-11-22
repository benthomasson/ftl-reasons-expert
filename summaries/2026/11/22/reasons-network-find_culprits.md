---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: find_culprits in reasons/network.py

# `find_culprits` — `Network.find_culprits`

## Purpose

Given a set of nogood node IDs (nodes that are simultaneously `IN`, which is a contradiction — the nogood), identify which **premises** (assumption nodes with no justifications) are candidate retraction targets for resolving that contradiction. Results are ranked so the *least entrenched* premise is first, protecting evidence and retracting speculation preferentially.

This is the core of the **dependency-directed backtracking** step in the TMS: instead of retracting an arbitrary node, it traces the contradiction back to the assumptions it rests on and picks the cheapest one to drop.

## Contract

- **Preconditions (soft):** `nogood_node_ids` normally refers to an actual active contradiction (all listed nodes `IN`). None are enforced — unknown IDs and non-`IN` nodes are silently skipped.
- **Postcondition:** Returns a list of dicts, one per candidate premise, sorted ascending by `entrenchment`.
- **Invariant:** Every element of each candidate's `would_resolve` list is a nogood node ID that (a) exists in `self.nodes` and (b) was `IN` at call time.
- **Invariant:** Every candidate's `premise` is a *premise* (a node with `justifications == []`), by virtue of `trace_assumptions`.
- **Assumption NOT enforced:** That retracting a returned `premise` will actually flip its `would_resolve` nodes to `OUT`. See “Assumptions” below.

## Parameters

- `nogood_node_ids: list[str]` — node IDs comprising the contradiction. Expected to be currently `IN`, but the function tolerates:
  - IDs absent from `self.nodes` → skipped
  - IDs with `truth_value != "IN"` → skipped
  - Empty list → returns `[]`

## Return Value

```python
[
  {
    "premise": str,            # premise node id
    "would_resolve": [str],    # nogood node ids whose assumption set contains this premise
    "dependent_count": int,    # len(premise_node.dependents) — raw fan-out
    "entrenchment": int,       # score from _entrenchment(premise_id); lower = retract first
  },
  ...
]
```

- Ascending by `entrenchment`: index `0` is the top retraction candidate.
- Caller (`add_nogood`) consumes `culprits[0]["premise"]`.
- Returns `[]` when no candidate premise is found (all nogood nodes missing/OUT, or no shared assumptions).

## Algorithm

1. **Collect assumptions per nogood node.**
   - For each `nid`:
     - Skip unknown IDs.
     - Skip nodes whose `truth_value != "IN"` — only `IN` nodes can be part of a live contradiction.
     - Call `trace_assumptions(nid)`, which recursively walks **every justification's antecedents** (not just the currently-valid one) down to nodes with empty `justifications` — that is, premises.
     - Store in `assumptions_by_node[nid]`; union into `all_premises`.

2. **Score each premise.**
   - For each `premise_id` in `all_premises`, build `would_resolve = [nid for nid in assumptions_by_node if premise_id in that list]`.
   - If non-empty, compute `entrenchment = self._entrenchment(premise_id)` and record `dependent_count = len(self.nodes[premise_id].dependents)`.

3. **Sort ascending by `entrenchment`.**

The `_entrenchment` score (in order of typical weight): `+100` for being a premise, `+50` source, `+25` source hash, `+10 × dependents`, plus a `beliefs_type` bonus (`AXIOM`/`WARNING` 90, `OBSERVATION` 80, `DERIVED` 40, `PREDICTED` 30, `NOTE` 10, default 20). Since every candidate is a premise, the `+100` is common overhead — the differentiators are source, source-hash, dependent fan-out, and metadata type.

## Side Effects

**None.** The function is a pure read/query: no mutation of nodes, justifications, dependents, log, or meta. It does not retract anything. Retraction is performed by `add_nogood` after this returns.

## Error Handling

- **No exceptions raised in the function body.** Inputs that would raise in a stricter design are silently filtered:
  - Unknown `nid` → `continue` before `trace_assumptions` is called.
  - `nid` not `IN` → `continue`.
- `self.nodes[premise_id]` is safe because `premise_id` came from `trace_assumptions`, which only added IDs present in `self.nodes`.
- `_entrenchment` itself returns `0` for unknown IDs (defensive, unused here).

## Usage Patterns

Primary caller is `add_nogood`:

```python
all_in = all(self.nodes[nid].truth_value == "IN" for nid in node_ids)
if not all_in:
    return []
culprits = self.find_culprits(node_ids)
if culprits:
    victim_id = culprits[0]["premise"]   # least entrenched
    ...
else:
    # fallback: retract nogood node with fewest dependents
```

Caller obligations:
- Pass in the *conflicting* node set (already verified `IN`), typically right after detecting a nogood.
- Treat the returned list as advisory ranking, not proof.
- Typically perform `retract(culprits[0]["premise"])` and let `_propagate` cascade.

## Dependencies

- `self.nodes: dict[str, Node]` — the belief store.
- `self.trace_assumptions(nid)` — reverse walk over justifications to premises (defined in the same class).
- `self._entrenchment(nid)` — entrenchment scoring (same class).
- `Node` fields: `justifications`, `dependents`, `truth_value`, `metadata`, `source`, `source_hash`.
- No external libraries; no I/O.

## Assumptions Not Enforced by Types

- **`would_resolve` is a superset, not a proof.** `trace_assumptions` walks *all* justifications of each node, including non-supporting ones. If a nogood node has an alternative valid justification that doesn't depend on `premise_id`, retracting `premise_id` will *not* flip it to `OUT` — yet the code still lists it in `would_resolve`. The docstring ("would cause at least one nogood node to go OUT") is stronger than what the code guarantees. In practice the ranking usually works because multiple valid justifications are rare, but this is a real semantic gap.
- **`dependent_count` is computed even for ghost premises.** The code assumes `self.nodes[premise_id].dependents` is up-to-date with justifications. If `_rebuild_dependents` hasn't been run after manual edits, this number can be stale.
- **No preference for `would_resolve` breadth.** A premise that would resolve *many* nogood nodes is treated the same as one that would resolve just one, except for its `dependent_count` bonus. This may under-retract narrowly-scoped assumptions.
- **The `+100` premise bonus flattens ranking among candidates.** All candidates are premises, so type/source/dependents dominate the tie-break. A `beliefs_type` in `metadata` can override dependent count (e.g. `AXIOM` 90 vs an extra dependent's `+10`), so type annotations on premises are load-bearing for the selection.

## Topics to Explore

- [function] `reasons/network.py:trace_assumptions` — How justification traversal collects premises; note it walks *all* justifications rather than only the currently-valid one, which underpins the `would_resolve` heuristic.
- [function] `reasons/network.py:_entrenchment` — The scoring breakdown that determines which premise `add_nogood` will retract; verify the type-score weights make sense for your ontology.
- [function] `reasons/network.py:add_nogood` — The only current caller; shows how `culprits[0]` is consumed and how the fallback path handles an empty result.
- [file] `tests/test_backtracking.py` — Behavioral coverage for the dependency-directed backtracking path, including expected retraction targets.
- [general] `doyle-tms-1979` — The classic Truth Maintenance System paper that this algorithm (nogood → backtrack to premises → retract) implements; useful for reconciling terminology like "premise", "justification", "entrenchment".

## Beliefs

- `find-culprits-is-pure` — `find_culprits` performs no mutation or logging and is callable safely as a read-only query.
- `find-culprits-skips-invalid-inputs` — Nodes not in `self.nodes` or with `truth_value != "IN"` are silently skipped rather than raising.
- `would-resolve-is-heuristic` — A premise appearing in `would_resolve` for a nogood node only means the premise is transitively in that node's assumption set; it does not guarantee retraction will flip the node to `OUT`.
- `find-culprits-returns-only-premises` — Every candidate's `premise` field points to a node with `justifications == []`, as guaranteed by `trace_assumptions`.
- `entrenchment-sorted-ascending` — Candidates are sorted by ascending `entrenchment`, so the least-protected assumption is at index 0.
- `add-nogood-uses-first-culprit` — `add_nogood` retracts `culprits[0]["premise"]` when the list is non-empty, else falls back to the nogood node with the fewest dependents.
