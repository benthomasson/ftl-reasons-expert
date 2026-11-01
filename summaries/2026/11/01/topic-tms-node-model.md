---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Topic: The core data model: nodes (IN/OUT), justifications (SL, CP, outlist), nogoods, and their relationships. Understood via `network.py`

# Core Data Model in `reasons/network.py`

## The Big Picture

This codebase implements **Doyle's (1979) Truth Maintenance System (TMS)** as a belief network. The fundamental idea: you have a graph of claims (nodes), each either **IN** (accepted) or **OUT** (retracted), and their truth values aren't stored independently — they're *derived* from justifications (rules) that link them together. When one node flips, a cascade ripples through the graph.

The `Network` class (line ~28) is the single container:

```python
self.nodes: dict[str, Node] = {}       # all claims
self.nogoods: list[Nogood] = []        # known contradictions
```

Everything else is methods on this graph.

---

## Nodes: The Atoms of Belief

A `Node` is a single claim. Its essential fields (visible in `add_node`, line ~90):

| Field | Meaning |
|-------|---------|
| `id` | String key, unique in the network |
| `text` | Human-readable statement of the belief |
| `truth_value` | `"IN"` or `"OUT"` — the only two states |
| `justifications` | List of rules that *can* make this node IN |
| `dependents` | **Reverse index**: set of node IDs whose truth depends on this node |
| `metadata` | Arbitrary tags (`access_tags`, `beliefs_type`, `_retracted`, etc.) |
| `source`, `source_hash` | Provenance — where the belief came from |

**Premises vs. derived nodes.** A node with `justifications == []` is a *premise* (line ~145 in `add_node`):

```python
if node.justifications:
    node.truth_value = self._compute_truth(node)
else:
    node.truth_value = "IN"  # premise — IN by default
```

This is the "evidence vs. speculation" distinction. Premises are things you *assert*; derived nodes are things you *infer*. The `_entrenchment` method (line ~253) encodes this: premises get +100, source-backed nodes get +50–75, and each dependent adds +10. This scoring is used by `find_culprits` to decide *which* premise to retract when a contradiction arises.

**Key invariant:** Retracted nodes are never deleted. From the header (line 3): *"Retracted nodes stay in the network (enables restoration without rederivation)."* This is why `assert_node` (line ~190) can simply flip `truth_value` back to `"IN"` and call `_propagate` — the node's justifications and dependents are still intact.

---

## Justifications: The Rules That Bind Nodes Together

A `Justification` is a rule that says "under these conditions, this node should be IN." It has two types:

### SL (Single Label / Simple Literal)

The simplest form. Seen in tests (`tests/test_network.py`, line ~22):

```python
Justification(type="SL", antecedents=["a"])
```

Semantics: **All** antecedents must be IN → node is IN. If **any** antecedent is OUT → node is OUT. No outlist needed. This is a pure positive dependency.

### CP (Compound / Conditional Positive)

The richer form. It carries both:
- **`antecedents`** — nodes that must be IN (positive conditions)
- **`outlist`** — nodes that must be OUT (negative conditions / "unless" clauses)

A CP justification is satisfied when *all* antecedents are IN **and** all outlist members are OUT. This lets you express "A and B together imply C, *unless* D is also true."

### Why Both `antecedents` and `outlist` Create Dependency Edges

Look at `_rebuild_dependents` (line ~38):

```python
for j in node.justifications:
    for ant_id in j.antecedents:
        if ant_id in self.nodes:
            self.nodes[ant_id].dependents.add(node.id)
    for out_id in j.outlist:
        if out_id in self.nodes:
            self.nodes[out_id].dependents.add(node.id)
```

Both directions matter. If node X is in Y's antecedents, then X changing affects Y. If node X is in Y's **outlist**, X changing *also* affects Y (because Y's "unless" condition may flip). The `dependents` set is the **reverse index** that makes propagation O(1) lookup rather than scanning all nodes.

This is the single most important structural relationship in the codebase: **every justification edge (whether antecedent or outlist) creates a directed dependency from the referenced node to the justifying node, tracked in `.dependents`.**

---

## Nogoods: Known Contradictions

A `Nogood` is a set of node IDs that **cannot all be IN simultaneously**. It's the formal representation of a contradiction.

- Stored as `self.nogoods: list[Nogood]` (line ~31).
- When a nogood is detected (all its members are IN), the system must retract at least one member to restore consistency.
- `find_culprits` (line ~290) takes a list of nogood node IDs and traces back through justification chains to find which *premises* to retract. It sorts candidates by entrenchment (lowest first), so speculative assumptions get retracted before evidence.

The nogood is the "problem" and the retraction cascade is the "solution mechanism."

---

## The Propagation Loop: How Truth Values Actually Change

The lifecycle of a state change:

1. **`retract(node_id)`** (line ~155) or **`assert_node(node_id)`** (line ~190) flips the node's `truth_value`.
2. Both call **`self._propagate(node_id)`** — a BFS/DFS over the `dependents` set.
3. For each dependent, `_compute_truth(node)` re-evaluates all justifications:
   - If **any** justification is satisfied (all ants IN, all outlist OUT) → node is IN.
   - If **no** justification is satisfied → node is OUT.
4. If the dependent's truth value changed, the cascade continues to *its* dependents.

This is visible in the test `test_retract_cascades_through_chain` (line ~72):

```
A → B → C    (each justified by SL with the previous as antecedent)
retract("a") → B goes OUT → C goes OUT
changed == {"a", "b", "c"}
```

The cascade is **transitive and automatic**. You never manually update B or C.

---

## The Relationship Graph (Mental Model)

```
Premise A (IN, no justifications)
    │
    │  SL justification: antecedents=[A]
    ▼
Derived B (IN, justified by A)
    │
    │  CP justification: antecedents=[B], outlist=[D]
    ▼
Derived C (IN iff B is IN AND D is OUT)
    
    D (IN or OUT — the "unless" condition)
    
Nogood: {C, E}  ← "C and E cannot both be IN"
```

- **Forward edges** (antecedent → dependent): "I support you."
- **Outlist edges** (outlist member → dependent): "I veto you."
- **Nogood edges**: "These nodes are mutually exclusive."
- **`dependents` reverse index**: the materialized adjacency list for fast propagation.

---

## What's Missing From These Observations

- **The `Node`, `Justification`, and `Nogood` class definitions** are imported from `reasons/__init__.py` (or a sibling module) — the actual dataclass/NamedTuple definitions aren't in the snippets.
- **`_compute_truth(node)`** — the actual satisfaction logic for SL vs. CP justifications — is referenced but not shown.
- **`_propagate(node_id)`** — the BFS/DFS implementation — is referenced but not shown.
- **CP justification in tests** — only SL is exercised in the visible test code; CP with outlist isn't demonstrated.
- **Nogood detection** — how the system *discovers* a nogood (vs. it being manually registered) isn't visible here.

To fully understand the model, you'd want to read the dataclass definitions and the `_compute_truth` / `_propagate` implementations.

---

## Topics to Explore

- [file] `reasons/__init__.py` — Contains (or re-exports) the `Node`, `Justification`, and `Nogood` class definitions; essential to see the exact field types and defaults
- [function] `reasons/network.py:_compute_truth` — The satisfaction logic that evaluates SL vs. CP justifications against current IN/OUT states; the heart of the TMS
- [function] `reasons/network.py:_propagate` — The BFS/DFS cascade that walks the `dependents` index and re-evaluates affected nodes
- [file] `tests/test_network.py` — Look for CP-justification tests (outlist behavior) and nogood tests to see the full contract in action
- [general] `doyle-tms-algorithm` — The 1979 paper this implements; reading the original clarifies *why* outlist exists and how "no-good" sets differ from simple negation

---

## Beliefs

- `node-truth-derives-from-justifications` — A node's `truth_value` is never stored independently of its justifications; it is always recomputed via `_compute_truth`, except for premises (no justifications) which default to IN
- `dependents-index-covers-outlist` — The `.dependents` reverse index is populated from *both* `antecedents` and `outlist` entries in a justification, not just antecedents; omitting outlist edges would break propagation of "unless" conditions
- `retract-preserves-node` — Retraction sets `truth_value = "OUT"` but never removes the node from `self.nodes`; this is a deliberate invariant that enables `assert_node` to restore a node without re-deriving its justifications
- `nogood-is-a-set-of-incompatible-inodes` — A Nogood represents a constraint that a set of node IDs cannot all simultaneously hold truth_value "IN"; resolving it requires at least one member to be retracted
- `entrenchment-ranks-premises-above-derived` — The `_entrenchment` scoring guarantees that premises (evidence, +100) are always preferred over derived nodes (speculation, +20–40) when `find_culprits` selects which nodes to retract
