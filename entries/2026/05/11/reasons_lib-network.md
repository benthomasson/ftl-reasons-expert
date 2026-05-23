# File: reasons_lib/network.py

**Date:** 2026-05-11
**Time:** 12:46

# `reasons_lib/network.py` — The TMS Dependency Network

## Purpose

This is the core engine of ftl-reasons. It implements Doyle's (1979) Truth Maintenance System (TMS) as an in-memory dependency graph where nodes carry truth values (`"IN"` or `"OUT"`) that propagate automatically when anything changes. Every other module in the project — storage, CLI, API, import, export — ultimately reads from or writes to an instance of `Network`.

The `Network` class owns three responsibilities:
1. **Graph management** — adding nodes, justifications, and nogoods
2. **Truth propagation** — computing truth values from justifications and cascading changes via BFS
3. **Conflict resolution** — dependency-directed backtracking when contradictions (nogoods) are detected

## Key Components

### `Network` (class)

The central data structure. State:
- `self.nodes: dict[str, Node]` — the belief graph, keyed by string ID
- `self.nogoods: list[Nogood]` — recorded contradictions
- `self.log: list[dict]` — timestamped audit trail of every propagation event
- `self.repos: dict[str, str]` — name-to-path mapping for multi-repo tracking

### Node lifecycle methods

| Method | What it does |
|--------|-------------|
| `add_node()` | Creates a node, registers it as a dependent of its antecedents/outlist, computes initial truth. Premises (no justifications) default to IN. |
| `retract(node_id, reason)` | Forces a node OUT, sets `_retracted` metadata flag, cascades via `_propagate()`. |
| `assert_node(node_id)` | Forces a node IN, removes `_retracted` flag, cascades. |
| `convert_to_premise(node_id)` | Strips all justifications, unlinks from antecedents' dependents sets, makes the node a premise (IN by default). |

### Justification management

| Method | Contract |
|--------|----------|
| `add_justification(node_id, j)` | Appends a justification, updates dependents index, recomputes truth, propagates. Returns old/new truth values and changed nodes. |
| `remove_justification(node_id, index)` | Removes one justification by index. Refuses if it's the last one (use `convert_to_premise` or `retract` instead). Cleans up dependents for references no longer used by remaining justifications. |

### Dialectical reasoning

| Method | What it does |
|--------|-------------|
| `challenge(target_id, reason)` | Creates a premise node and adds it to the target's outlist. Since the challenge is IN, the target goes OUT. If the target was a premise, it's converted to a justified node with the challenge in its outlist. |
| `defend(target_id, challenge_id, reason)` | Challenges the challenge — creates a defense node that puts the challenge in its outlist, causing the challenge to go OUT, which restores the target. Implemented by delegating to `challenge()`. |
| `supersede(old_id, new_id)` | Adds `new_id` to `old_id`'s outlist. When `new_id` is IN, `old_id` goes OUT. Reversible — retracting `new_id` restores `old_id`. |

### Conflict resolution

| Method | What it does |
|--------|-------------|
| `add_nogood(node_ids)` | Records a contradiction. If all nodes are IN, uses `find_culprits()` to identify the least-entrenched premise, then retracts it. |
| `find_culprits(node_ids)` | Traces each nogood node back to its premises via `trace_assumptions()`, scores them by `_entrenchment()`, returns candidates sorted least-entrenched first. |
| `_entrenchment(node_id)` | Scoring function: premises +100, sourced +50/+25, dependents +10 each, type-based (AXIOM 90, OBSERVATION 80, DERIVED 40, PREDICTED 30, NOTE 10). |

### Introspection

| Method | Returns |
|--------|---------|
| `explain(node_id)` | List of explanation steps tracing back through justification chains. For IN nodes, finds the valid justification and recurses into antecedents. For OUT nodes, lists which antecedents failed and which outlist items are violated. |
| `trace_assumptions(node_id)` | All premise IDs in the backward dependency subgraph. |
| `trace_access_tags(node_id)` | Union of `access_tags` metadata across the entire backward subgraph. |
| `get_belief_set()` | All node IDs currently IN. |
| `verify_dependents()` | Consistency check — compares the live dependents index against what justifications imply. Returns error strings. |

## Patterns

**BFS propagation.** `_propagate()` uses a `deque` for breadth-first traversal of dependents. It skips nodes marked `_retracted` and logs dangling dependents as warnings rather than raising.

**Disjunctive justification semantics.** `_compute_truth()` returns IN if *any* justification is valid (disjunction). Each individual justification requires *all* antecedents IN and *all* outlist members OUT (conjunction). This is the SL justification from Doyle's original paper.

**Outlist for non-monotonic reasoning.** The outlist mechanism ("believe X unless Y") powers supersession, challenges, and GATE beliefs. When an outlist node goes IN, the justified node goes OUT — and vice versa.

**Dependents as a reverse index.** Each node maintains a `dependents: set[str]` that tracks which other nodes have it in their antecedents or outlist. This is a manually maintained reverse index — `_rebuild_dependents()` can reconstruct it from scratch, and `verify_dependents()` can audit it.

**Metadata as an extension point.** The `metadata` dict on each node carries `_retracted`, `retract_reason`, `superseded_by`, `supersedes`, `challenges`, `access_tags`, `summarized_by`, and `beliefs_type` — all used by different methods without a formal schema.

## Dependencies

**Imports:** `Node`, `Justification`, `Nogood` from `reasons_lib.__init__` (data classes), plus stdlib `deque` and `datetime`.

**Imported by:** This is the most-imported module in the project. Direct consumers include `storage.py` (serialization/deserialization), `api.py` (HTTP/CLI interface), `compact.py` (summarization), `check_stale.py` (staleness detection), `import_beliefs.py` and `import_agent.py` (bulk import), and `export_markdown.py`. Nearly every test file imports `Network` directly.

## Flow

A typical truth propagation cycle:

1. Caller invokes `retract("node-a")` or `assert_node("node-a")`
2. The node's truth value is set directly
3. `_propagate("node-a")` is called — BFS through `node-a.dependents`
4. For each dependent, `_compute_truth()` checks all justifications
5. If truth value changed, the dependent is enqueued for further propagation
6. Every step is recorded in `self.log` via `_log()`

For nogood resolution:

1. `add_nogood(["x", "y"])` records the contradiction
2. If both x and y are IN, `find_culprits()` traces back to premises
3. `_entrenchment()` scores each premise
4. The least-entrenched premise is retracted, cascading through the graph

## Invariants

- **Node IDs are unique.** `add_node()` raises `ValueError` if the ID already exists.
- **All referenced nodes must exist** for `retract`, `assert_node`, `add_nogood`, `explain`, etc. — they raise `KeyError` otherwise.
- **A justified node with one justification cannot have it removed** via `remove_justification()` — must use `convert_to_premise` or `retract` instead.
- **Dependents index must stay consistent** with justifications. `_rebuild_dependents()` is the canonical reconstruction; `verify_dependents()` audits for drift.
- **Retracted nodes stay in the graph.** `retract()` sets `_retracted` metadata but never removes the node, enabling later restoration without rederivation.
- **`_propagate` skips `_retracted` nodes.** A retracted node's truth value won't be overwritten by propagation — it stays OUT until explicitly asserted.

## Error Handling

All errors are immediate exceptions — no error codes or result objects:
- `ValueError` for duplicate node IDs, challenge IDs, and attempts to remove the last justification
- `KeyError` for references to nonexistent nodes
- `IndexError` for out-of-range justification indices

Dangling dependents (a dependent ID that doesn't exist in `self.nodes`) are logged as warnings via `_log("warn", ...)` during propagation but don't raise. This is a conscious design choice — the graph tolerates stale dependents references.

## Topics to Explore

- [file] `reasons_lib/storage.py` — Serializes and deserializes `Network` to/from SQLite and JSON; understanding the persistence layer explains how the in-memory graph survives across CLI invocations
- [file] `reasons_lib/__init__.py` — Defines the `Node`, `Justification`, and `Nogood` data classes that `Network` operates on
- [function] `reasons_lib/compact.py:compact` — Uses `summarize()` to collapse groups of nodes into summary nodes for token-budget-constrained output
- [file] `tests/test_network.py` — The primary test suite; exercises add/retract/propagate/nogood/explain paths and documents expected behavior at the edge cases
- [general] `outlist-dependents-tracking` — The CLAUDE.md documents a known issue where outlist nodes aren't automatically re-evaluated when their truth value changes; understanding this gap is critical for maintaining GATE beliefs

## Beliefs

- `network-disjunctive-justification` — A node is IN if ANY of its justifications is valid; each justification requires ALL antecedents IN and ALL outlist members OUT
- `network-retracted-nodes-persist` — Retracted nodes remain in `self.nodes` with `_retracted` metadata set; they are never deleted, enabling restoration without rederivation
- `network-propagate-skips-retracted` — BFS propagation in `_propagate()` skips nodes whose metadata contains `_retracted=True`, preventing propagation from overriding an explicit retraction
- `network-entrenchment-prefers-premises` — The entrenchment scoring function gives premises +100 over derived nodes, ensuring `find_culprits()` retracts speculative assumptions before evidence
- `network-single-justification-removal-blocked` — `remove_justification()` raises `ValueError` when a node has exactly one justification, forcing callers to use `convert_to_premise` or `retract` instead

