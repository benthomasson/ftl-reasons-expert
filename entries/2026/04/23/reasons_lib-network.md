# File: reasons_lib/network.py

**Date:** 2026-04-23
**Time:** 16:33

---

## Purpose

`network.py` is the core engine of the Reason Maintenance System (RMS). It owns the dependency graph of beliefs and the algorithm that propagates truth values through that graph. Every other module in `reasons_lib` — storage, import, export, staleness checking, compaction — operates on or around the `Network` object this file defines.

The implementation follows Doyle's (1979) Truth Maintenance System (TMS) design: nodes represent beliefs, justifications encode why a belief is held, and truth values (`IN`/`OUT`) propagate automatically when anything changes. The key property is **non-monotonic reasoning** — beliefs can be retracted and restored, and those changes cascade through the dependency graph.

## Key Components

### `Network` class

The single class in the file. Holds:

- **`nodes`** — `dict[str, Node]`: the belief graph, keyed by node ID.
- **`nogoods`** — `list[Nogood]`: recorded contradictions (sets of nodes that can't all be IN).
- **`repos`** — `dict[str, str]`: name-to-path mapping for tracked repositories.
- **`log`** — `list[dict]`: audit trail of every propagation event (action, target, value, timestamp).

### Core operations

| Method | What it does | Returns |
|--------|-------------|---------|
| `add_node` | Insert a new node, register it as a dependent of its antecedents/outlist nodes, compute initial truth value | `Node` |
| `retract` | Force a node OUT, mark `_retracted` in metadata, cascade | `list[str]` of changed IDs |
| `assert_node` | Force a node IN, remove `_retracted` flag, cascade | `list[str]` of changed IDs |
| `add_justification` | Append a new justification to an existing node, recompute, propagate | `dict` with old/new truth values and changed list |
| `supersede` | Wire `new_id` into `old_id`'s outlist so old goes OUT when new is IN (reversible) | `dict` |
| `challenge` | Create a new premise node and add it to the target's outlist, driving the target OUT | `dict` |
| `defend` | Counter-challenge: creates a defense node that challenges the challenge, restoring the target | `dict` |
| `add_nogood` | Record a contradiction, then use dependency-directed backtracking to retract the least-entrenched responsible premise | `list[str]` of changed IDs |
| `summarize` | Create a derived node that is IN when all summarized nodes are IN | `dict` |
| `convert_to_premise` | Strip justifications, making a derived node into an unconditional premise | `dict` |

### Query operations

| Method | What it does |
|--------|-------------|
| `trace_assumptions` | DFS backward through justification chains to find all premise ancestors |
| `explain` | Produce a human-readable trace of why a node is IN or OUT |
| `get_belief_set` | Return all node IDs currently IN |
| `find_culprits` | For a set of contradicting nodes, rank premises by entrenchment to find the best retraction candidate |

### Internal machinery

| Method | Role |
|--------|------|
| `_propagate` | BFS through `dependents` links, recomputing truth values at each hop |
| `_compute_truth` | Node is IN if ANY justification is valid; premise keeps its current value |
| `_justification_valid` | SL/CP: all antecedents must be IN AND all outlist nodes must be OUT |
| `_entrenchment` | Scoring heuristic for how "protected" a node is (premises > derived, sourced > unsourced, many dependents > few) |
| `_log` | Append to the audit trail |
| `recompute_all` | Fixpoint iteration over every derived node — used after bulk operations like import |

## Patterns

**Non-monotonic justification via outlist.** The outlist is the mechanism that makes this a TMS rather than a simple DAG. A justification is valid only when all antecedents are IN *and* all outlist nodes are OUT. This single rule powers supersession, challenges, defenses, and default reasoning ("believe X unless Y").

**Bidirectional indexing.** Each node tracks its `dependents` — the reverse of the antecedent/outlist edges. This is maintained eagerly in `add_node`, `add_justification`, `supersede`, `challenge`, and `convert_to_premise`. The forward direction (justification → antecedent) is walked for `trace_assumptions` and `explain`; the reverse (`dependents`) is walked for `_propagate`.

**BFS propagation.** `_propagate` uses a `deque`-based BFS, not DFS. This gives breadth-first wavefront expansion, which is the standard approach for TMS propagation — it processes nodes at each "distance" from the change before moving deeper.

**Dialectical layering.** `challenge` and `defend` are both built on the same outlist mechanism. A defense is literally a challenge *of the challenge*. This recursive structure means you can have chains of challenges and defenses, and the truth values resolve automatically.

**Dependency-directed backtracking.** `add_nogood` doesn't just pick a random node to retract. It traces back to premises via `trace_assumptions`, scores them with `_entrenchment`, and retracts the least-entrenched one — minimizing disruption to the belief set.

**Fixpoint iteration.** `recompute_all` loops until no truth values change, bounded by `len(nodes) + 1` iterations. This handles cascading dependencies that might not resolve in a single pass due to arbitrary node ordering.

## Dependencies

**Imports:**
- `collections.deque` — BFS queue for `_propagate`
- `datetime` — timestamps for `_log` and `Nogood.discovered`
- `Node`, `Justification`, `Nogood` from `reasons_lib/__init__.py` — the three dataclasses that define the data model

**Imported by:** Essentially everything else in the project. It's the central data structure. `api.py` wraps it for the CLI, `storage.py` serializes/deserializes it, `import_beliefs.py` and `import_agent.py` populate it, `export_markdown.py` renders it, `compact.py` prunes it, `check_stale.py` validates it. All test files exercise it directly.

## Flow

### Adding a node

1. Validate ID uniqueness → raise `ValueError` if duplicate.
2. Create `Node` dataclass.
3. Walk all justifications' antecedents and outlist, adding this node's ID to each referenced node's `dependents` set.
4. Register node in `self.nodes`.
5. If the node has justifications, compute truth via `_compute_truth`. Otherwise it's a premise → `IN`.
6. Log the addition.

### Retraction cascade

1. `retract(node_id)` sets the node OUT, marks `_retracted` in metadata.
2. Calls `_propagate(node_id)`.
3. `_propagate` does BFS over `dependents`. For each dependent not yet visited and not manually retracted: recompute truth value. If changed, record it, add to BFS queue.
4. Returns the full list of changed node IDs.

### Nogood resolution

1. `add_nogood` records the contradiction.
2. Checks if all nogood nodes are currently IN (active contradiction).
3. If active: `find_culprits` traces each nogood node back to its premises, scores them by `_entrenchment`.
4. Retracts the least-entrenched premise (the one most likely to be speculative rather than evidential).
5. If no premises found (shouldn't happen normally), falls back to retracting the nogood node with fewest dependents.

### Challenge/defend cycle

1. `challenge(target, reason)` creates a new premise node (IN) and adds it to every justification's outlist on the target. Target recomputes → goes OUT (because outlist node is IN).
2. `defend(target, challenge, reason)` calls `challenge(challenge_id, reason)` — it challenges the challenge. The defense node goes into the challenge's outlist. Challenge goes OUT → target's outlist clears → target comes back IN.

## Invariants

1. **Node IDs are unique.** `add_node` raises `ValueError` on duplicate. Challenge/defense ID generation auto-suffixes to avoid collisions.
2. **Dependents index is always consistent with justifications.** Every method that modifies justifications (`add_node`, `add_justification`, `supersede`, `challenge`, `convert_to_premise`) updates `dependents` accordingly. `convert_to_premise` uses `discard` to clean up.
3. **A premise (no justifications) defaults to IN.** `_compute_truth` preserves the current value for unjustified nodes. `add_node` sets `IN` for new premises.
4. **A derived node is IN iff at least one justification is valid** (disjunctive semantics). This is the TMS well-founded semantics rule.
5. **A justification is valid iff all antecedents are IN and all outlist nodes are OUT.** Missing antecedents (not in network) fail the check. Missing outlist nodes (not in network) pass — the "open world" default.
6. **Retracted nodes (`_retracted` metadata) are skipped during propagation** but remain in the network for potential restoration.
7. **`recompute_all` terminates** — bounded by `len(nodes) + 1` iterations, and monotonically converges since each pass can only change nodes whose inputs changed.

## Error Handling

- **`KeyError`** raised by `retract`, `assert_node`, `trace_assumptions`, `explain`, `supersede`, `add_justification`, `challenge`, `defend`, `convert_to_premise`, `summarize`, `add_nogood` when a referenced node doesn't exist.
- **`ValueError`** raised by `add_node` and `challenge` when a node ID already exists.
- No exceptions are caught within this module — all errors propagate to the caller (typically `api.py`).
- Dangling references in justifications (antecedent/outlist IDs not in the network) are handled gracefully: `_justification_valid` treats missing antecedents as failing (node goes OUT) and missing outlist as passing (doesn't block). `add_node` silently skips registration for missing antecedents/outlist when building the `dependents` index.

## Topics to Explore

- [file] `reasons_lib/storage.py` — How the Network is serialized to/from SQLite and JSON; the persistence boundary for this in-memory graph
- [file] `reasons_lib/api.py` — The CLI-facing wrapper that orchestrates Network operations and bridges to storage
- [function] `reasons_lib/compact.py:compact` — How nodes are pruned from the network while preserving semantic integrity (interacts with `summarize` and `recompute_all`)
- [file] `tests/test_dialectical.py` — Test cases for the challenge/defend cycle; the best way to understand the dialectical layering in practice
- [general] `doyle-1979-tms` — The original paper "A Truth Maintenance System" by Jon Doyle; understanding SL-justifications and the outlist mechanism at the theoretical level clarifies every design choice in this file

## Beliefs

- `network-disjunctive-justification` — A derived node is IN if ANY of its justifications is valid (disjunctive semantics), not all of them
- `network-outlist-asymmetry` — Missing antecedents cause a justification to fail (closed-world), but missing outlist nodes are treated as OUT (open-world), creating an asymmetry in how dangling references behave
- `network-retracted-nodes-persist` — Retracted nodes remain in the network with `_retracted` metadata; they are skipped during propagation but can be restored via `assert_node`
- `network-entrenchment-scoring` — Entrenchment is a heuristic sum: premises +100, source +50, source_hash +25, dependents ×10, plus type-based scores (AXIOM 90 down to NOTE 10); used only by `find_culprits` for nogood resolution
- `network-defense-is-recursive-challenge` — `defend` is implemented as `challenge(challenge_id, ...)`, meaning the entire dialectical hierarchy uses a single mechanism (outlist insertion) at every level

