# File: reasons_lib/network.py

**Date:** 2026-05-05
**Time:** 15:31

# `reasons_lib/network.py` — The Dependency Network

## Purpose

This is the core engine of the entire `ftl-reasons` project. It implements Doyle's (1979) Truth Maintenance System (TMS) as a single `Network` class that owns the dependency graph of beliefs, their justifications, and the propagation of truth values through that graph.

Everything else in the codebase — storage, CLI, import/export, LLM-driven derivation — is plumbing around this file. `Network` is the only place where truth values are computed and propagated. If you understand this file, you understand the system.

## Key Components

### `Network` class

The sole class. Holds the graph in-memory as a `dict[str, Node]` plus a list of `Nogood` contradictions. Key state:

- **`self.nodes`** — the belief graph. Keys are kebab-case string IDs; values are `Node` objects (defined in `__init__.py`).
- **`self.nogoods`** — recorded contradictions between sets of nodes.
- **`self.log`** — append-only audit trail of every propagation event (add, retract, propagate, backtrack, etc.).
- **`self.repos`** — name-to-path mapping for multi-repo tracking (not used by the core algorithm).

### Core operations

| Method | What it does |
|---|---|
| `add_node()` | Insert a node, wire up dependents index, compute initial truth value, log it. |
| `retract(node_id, reason)` | Force a node OUT, mark `_retracted` in metadata, cascade via `_propagate()`. |
| `assert_node(node_id)` | Force a node IN, clear `_retracted`, cascade via `_propagate()`. |
| `add_justification()` | Attach a new justification to an existing node, recompute, propagate. |
| `recompute_all()` | Fixpoint iteration over all derived nodes — used after bulk imports. |

### Non-monotonic reasoning

| Method | What it does |
|---|---|
| `supersede(old, new)` | Adds `new` to `old`'s outlist so `old` goes OUT when `new` is IN. Reversible. |
| `challenge(target, reason)` | Creates a premise node and adds it to the target's outlist, forcing the target OUT. |
| `defend(target, challenge, reason)` | Challenges the challenge — creates a defense premise that puts the challenge in *its* outlist, restoring the target. |

### Contradiction resolution

| Method | What it does |
|---|---|
| `add_nogood(node_ids)` | Records a contradiction, then uses dependency-directed backtracking to retract the least-entrenched premise. |
| `find_culprits(node_ids)` | Traces each nogood node back to its premises, scores them by entrenchment, returns candidates sorted least-entrenched-first. |
| `_entrenchment(node_id)` | Heuristic score: premises > derived, sourced > unsourced, more dependents > fewer. Protects evidence, sacrifices speculation. |

### Query / introspection

| Method | What it does |
|---|---|
| `explain(node_id)` | Recursive trace producing a step-by-step explanation of why a node is IN or OUT. |
| `trace_assumptions(node_id)` | Walks backward through justification chains to find all premise nodes supporting a conclusion. |
| `trace_access_tags(node_id)` | Collects `access_tags` from the entire dependency subgraph. |
| `get_belief_set()` | Returns all node IDs currently IN. |
| `verify_dependents()` | Consistency check — recomputes the dependents index from justifications and diffs against the live index. |

## Patterns

**BFS propagation.** `_propagate()` uses a `deque`-based BFS through the `dependents` reverse index. When a node's truth value changes, its dependents are recomputed. If they change too, their dependents are enqueued. The `visited` set prevents cycles.

**Fixpoint iteration.** `recompute_all()` loops over all derived nodes until no truth values change in a pass, bounded by `len(self.nodes) + 1` iterations. This handles cases where BFS ordering misses cascading dependencies.

**Reverse index maintenance.** Every mutation that touches justifications (`add_node`, `add_justification`, `supersede`, `challenge`, `convert_to_premise`) manually updates `node.dependents` for affected antecedent and outlist nodes. `_rebuild_dependents()` is the nuclear option that recomputes the entire index from scratch.

**Outlist-based defaults.** The SL justification semantics — "all antecedents IN AND all outlist OUT" — is the mechanism behind `supersede`, `challenge`, and `defend`. These are all syntactic sugar over outlist manipulation.

**Metadata-as-side-channel.** Retraction state (`_retracted`), supersession, challenges, summaries, and access tags are all stored in `node.metadata`. The core algorithm only looks at `_retracted`; the rest is for display and querying.

## Dependencies

**Imports:** Minimal — `deque` for BFS, `datetime` for timestamps, and the project's own data classes (`Node`, `Justification`, `Nogood`) from `reasons_lib/__init__.py`.

**Imported by:** Nearly everything. The `Network` class is used by:
- `storage.py` — serialization to/from SQLite
- `api.py` — HTTP/CLI interface layer
- `compact.py`, `export_markdown.py`, `import_beliefs.py`, `import_agent.py`, `check_stale.py` — various transformations
- 18 test files covering every public method

## Flow

A typical lifecycle:

1. **Bootstrap:** `storage.py` loads nodes from SQLite, creates a `Network`, calls `_rebuild_dependents()` and `recompute_all()`.
2. **Mutation:** User adds/retracts/challenges a node via `api.py`, which calls the corresponding `Network` method.
3. **Propagation:** The mutation method calls `_propagate()`, which BFS-walks dependents, calling `_compute_truth()` on each. Changed nodes are enqueued for further propagation.
4. **Truth computation:** `_compute_truth()` checks each justification via `_justification_valid()`. A node is IN if *any* justification is valid (disjunctive semantics).
5. **Persistence:** `storage.py` writes the updated network back to SQLite.

The critical path is: mutation → `_propagate()` → `_compute_truth()` → `_justification_valid()`. Everything else is bookkeeping.

## Invariants

1. **A premise with no justifications is IN by default** — `add_node()` sets `truth_value = "IN"` when `justifications` is empty.
2. **A derived node is IN iff at least one justification is valid** — enforced by `_compute_truth()`.
3. **SL justification validity requires all antecedents IN and all outlist OUT** — enforced by `_justification_valid()`.
4. **Retracted nodes (`_retracted` in metadata) are skipped during propagation** — `_propagate()` checks this before recomputing.
5. **Node IDs are unique** — `add_node()` raises `ValueError` on duplicate.
6. **The dependents index must mirror justifications** — `verify_dependents()` can detect drift; `_rebuild_dependents()` can repair it.
7. **Nogoods are monotonically numbered** — `_next_nogood_id` increments and is never reused.
8. **`recompute_all()` terminates** — bounded by `len(self.nodes) + 1` iterations.

## Error Handling

Straightforward: `KeyError` for missing nodes, `ValueError` for duplicates. No try/except blocks anywhere — errors propagate to the caller. The `_propagate()` method logs a warning for dangling dependents (a dependent ID that isn't in `self.nodes`) but doesn't raise, preventing a corrupt index from crashing propagation.

There is no validation that justification antecedents/outlist refer to existing nodes at add time — missing references are silently skipped in `_justification_valid()` (antecedents treated as not-IN, outlist treated as OUT). This is by design: it allows forward-references during bulk import, resolved later by `recompute_all()`.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Defines `Node`, `Justification`, and `Nogood` data classes that `Network` operates on
- [file] `reasons_lib/storage.py` — How the in-memory `Network` is serialized to/from SQLite, including `_rebuild_dependents()` calls on load
- [function] `reasons_lib/compact.py:compact` — How summary nodes are used to reduce token budget while preserving the dependency graph
- [file] `tests/test_network.py` — Canonical tests for propagation, retraction cascades, and outlist semantics
- [general] `doyle-1979-tms` — The original paper describing the SL justification semantics and dependency-directed backtracking that this code implements

## Beliefs

- `network-is-sole-truth-propagation-engine` — All truth value computation and propagation in the system flows through `Network._propagate()` and `Network._compute_truth()`; no other module modifies truth values directly.
- `sl-justification-is-disjunctive` — A node is IN if *any* of its justifications is valid, not all; `_compute_truth()` short-circuits on the first valid justification.
- `outlist-semantics-enable-nonmonotonic-reasoning` — `supersede`, `challenge`, and `defend` are all implemented as outlist manipulation on SL justifications, not as separate mechanisms.
- `retracted-nodes-skip-propagation` — `_propagate()` checks `metadata["_retracted"]` and skips recomputation, meaning manually retracted nodes are "pinned OUT" even if their justifications become valid.
- `forward-references-are-silently-tolerated` — `_justification_valid()` treats missing antecedent nodes as not-IN and missing outlist nodes as OUT, allowing nodes to be added before their dependencies exist.

