# File: reasons_lib/network.py

**Date:** 2026-04-24
**Time:** 16:47

# `reasons_lib/network.py` — The Dependency Network

## Purpose

This is the core engine of the entire system. `Network` owns the belief graph: it stores nodes, manages their justifications, computes truth values, and propagates changes. Every other module — storage, CLI, API, import/export — is a client of this class. If you understand this file, you understand how the TMS works.

It implements Doyle's 1979 Truth Maintenance System algorithm: nodes hold beliefs, justifications encode why those beliefs are held, and truth values propagate automatically when anything changes. The network handles both retraction cascades (a node goes OUT, dependents follow) and restoration (a node comes back IN, dependents recompute).

## Key Components

### `Network` class

The single class in the file. Holds all state:

- **`nodes: dict[str, Node]`** — The belief graph. Keyed by node ID.
- **`nogoods: list[Nogood]`** — Recorded contradictions.
- **`log: list[dict]`** — Append-only audit trail of every propagation event.
- **`repos: dict[str, str]`** — Name-to-path mapping for source tracking (used by staleness checking).

### Node lifecycle methods

| Method | What it does | Returns |
|---|---|---|
| `add_node()` | Creates a node, wires up dependents index, computes initial truth | The new `Node` |
| `retract()` | Forces a node OUT, sets `_retracted` metadata, cascades | List of changed IDs |
| `assert_node()` | Forces a node IN, clears `_retracted`, cascades | List of changed IDs |
| `add_justification()` | Adds a new justification to an existing node, re-propagates | Dict with old/new truth + changed |
| `convert_to_premise()` | Strips all justifications, making a derived node into a premise | Dict with changed |

### Non-monotonic reasoning

- **`challenge()`** — Creates a new premise node and adds it to the target's outlist. Since the challenge is IN, the target's SL justification becomes invalid (outlist member is IN), so the target goes OUT. This is the "believe X unless Y" mechanism applied dialectically.

- **`defend()`** — Implemented as `challenge(challenge_id, ...)` — a defense is just a challenge to the challenge. Elegant recursion of the same mechanism: defense goes IN → challenge goes OUT → target's outlist clears → target comes back IN.

- **`supersede()`** — Adds the new node to the old node's outlist. Same mechanism as challenge but with different metadata semantics. Reversible: retract the new node and the old one restores.

### Contradiction handling

- **`add_nogood()`** — Records a contradiction (a set of nodes that can't all be IN), then uses dependency-directed backtracking to resolve it. Calls `find_culprits()` to trace back to premises, then retracts the least-entrenched one.

- **`find_culprits()`** — For each node in the nogood set, traces back to its premises via `trace_assumptions()`. Ranks premises by entrenchment score (least entrenched first = retract speculative assumptions before evidence).

- **`_entrenchment()`** — Scoring heuristic: premises get +100, source-backed nodes get +50/+25, dependents count ×10, and metadata type (AXIOM=90, OBSERVATION=80, DERIVED=40, etc.). This protects evidence and axioms while sacrificing speculation.

### Query methods

- **`explain()`** — Recursive trace producing a list of explanation steps. For IN nodes, finds the valid justification and recurses into antecedents. For OUT nodes, lists which antecedents failed and which outlist members are violated.
- **`trace_assumptions()`** — DFS backward through justification chains to find all premise nodes a conclusion rests on.
- **`trace_access_tags()`** — Same backward walk but collecting `access_tags` metadata for provenance filtering.
- **`get_belief_set()`** — Simple filter: all node IDs where `truth_value == "IN"`.
- **`recompute_all()`** — Fixpoint iteration over all derived nodes. Keeps recomputing until no truth values change. Bounded by `len(nodes) + 1` iterations.

## Patterns

**BFS propagation.** `_propagate()` uses a `deque`-based BFS through the `dependents` reverse index. This ensures all affected nodes are recomputed in breadth-first order — important because a node's truth depends on its antecedents, and BFS processes closer dependencies first.

**Dependents as a reverse index.** Each node has a `dependents: set[str]` field that tracks which other nodes reference it (in their antecedents or outlist). This is maintained eagerly on `add_node()`, `add_justification()`, `challenge()`, etc. The `_rebuild_dependents()` method reconstructs it from scratch for consistency, and `verify_dependents()` can audit it.

**Uniform challenge/defense/supersede mechanism.** All three use the same outlist trick: add a node to another's outlist so that when it's IN, the target goes OUT. The differences are purely in metadata. This is a clean application of Doyle's non-monotonic justification formalism.

**Explicit retraction vs. computed truth.** There's a subtle distinction: `retract()` forces a node OUT and sets `_retracted` metadata. `_propagate()` skips `_retracted` nodes — they stay OUT regardless of what their justifications say. `assert_node()` clears `_retracted` and forces IN. Derived nodes without `_retracted` have their truth computed from justifications.

**ANY-mode justification semantics.** `_compute_truth()` returns IN if *any* justification is valid. This means a node with multiple justifications is resilient — it stays IN as long as at least one justification holds. This is standard TMS behavior.

## Dependencies

**Imports:** Only three — `deque` for BFS, `datetime` for audit log timestamps, and the project's own dataclasses (`Node`, `Justification`, `Nogood` from `__init__.py`). No external dependencies. This is deliberate: the core algorithm is pure logic.

**Imported by:** Everything. This is the foundational module. `api.py` wraps it in a functional interface, `storage.py` serializes/deserializes it, `compact.py` reads its structure for summarization, `import_beliefs.py` populates it, `export_markdown.py` reads it, and every test file exercises it. It imports nothing from the rest of the project.

## Flow

A typical lifecycle:

1. **Add premises**: `add_node("A", "some fact")` — no justifications, truth = IN.
2. **Add derived**: `add_node("B", "conclusion", justifications=[SL(antecedents=["A"])])` — truth computed from A being IN → B is IN.
3. **Challenge**: `challenge("A", "maybe not")` → challenge node created (IN), A gets outlist entry → A goes OUT → `_propagate(A)` recomputes B → B goes OUT.
4. **Defend**: `defend("A", "challenge-A", "actually yes")` → defense challenges the challenge → challenge goes OUT → A's outlist clears → A comes back IN → `_propagate` → B comes back IN.

The propagation flow: any method that changes a truth value calls `_propagate(changed_id)`. `_propagate` does BFS through `dependents`, calling `_compute_truth()` on each. If truth changed, it's added to the queue. The method returns all changed IDs so callers can report what was affected.

## Invariants

1. **Node IDs are unique.** `add_node()` raises `ValueError` if the ID exists.
2. **Dependents index mirrors justifications.** Every antecedent and outlist reference creates a reverse entry in `dependents`. `verify_dependents()` can audit this.
3. **Premises are IN by default.** A node with no justifications gets `truth_value = "IN"` on creation.
4. **`_retracted` nodes are frozen.** `_propagate()` skips nodes with `_retracted` metadata — their truth isn't recomputed from justifications. Only `assert_node()` can un-retract them.
5. **SL validity = all antecedents IN AND all outlist OUT.** This is the core invariant of the TMS. `_justification_valid()` enforces it. Missing nodes in antecedents are treated as OUT (fail); missing nodes in outlist are treated as OUT (pass).
6. **Recompute terminates.** `recompute_all()` bounds iterations to `len(nodes) + 1`, preventing infinite loops in pathological graphs.

## Error Handling

Straightforward: `KeyError` for missing nodes, `ValueError` for duplicate nodes. No error swallowing — every invalid operation raises immediately. The `_propagate()` method handles one edge case gracefully: dangling dependents (a node references a dependent that no longer exists) emit a log warning instead of crashing.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The `Node`, `Justification`, and `Nogood` dataclasses that define the graph's data model
- [function] `reasons_lib/storage.py:save` — How the in-memory `Network` is serialized to SQLite, especially justification and metadata round-tripping
- [file] `reasons_lib/compact.py` — Token-budgeted summarization that leverages summary nodes to compress network state
- [function] `reasons_lib/api.py:add_node` — The functional API layer that wraps `Network` methods and returns plain dicts for CLI/LangGraph consumption
- [general] `outlist-semantics` — How the outlist mechanism unifies challenge, defense, and supersession into one primitive

## Beliefs

- `network-any-mode-justification` — A node is IN if ANY of its justifications is valid; it only goes OUT when ALL justifications are invalid
- `network-retracted-skips-propagation` — Nodes with `_retracted` metadata are skipped during `_propagate()` and remain OUT regardless of justification validity until explicitly asserted
- `network-no-external-deps` — `network.py` depends only on stdlib (`deque`, `datetime`) and project dataclasses; it has zero external package dependencies
- `network-dependents-eagerly-maintained` — The `dependents` reverse index is updated in every method that modifies justifications (`add_node`, `add_justification`, `challenge`, `supersede`, `convert_to_premise`), not lazily recomputed
- `network-missing-outlist-passes` — In `_justification_valid()`, a node ID in the outlist that doesn't exist in the network is treated as OUT (the condition passes), while a missing antecedent is treated as not-IN (the condition fails)

