# File: reasons_lib/network.py

**Date:** 2026-04-29
**Time:** 16:52

# `reasons_lib/network.py` — The Dependency Network

## Purpose

This is the core engine of the entire TMS (Truth Maintenance System). It owns one responsibility: **maintaining a graph of beliefs where truth values propagate automatically**. Every other module in `reasons_lib` either feeds data into this network or reads state out of it.

The `Network` class implements Doyle's 1979 TMS algorithm: nodes hold beliefs, justifications encode dependencies between them, and when any node's truth value changes, the consequences ripple through the graph via BFS propagation. The key insight is that nodes are never deleted — retraction marks them OUT, which means restoration is possible without re-deriving anything.

## Key Components

### `Network` (class)

The central data structure. Holds:
- `nodes: dict[str, Node]` — the belief graph, keyed by ID
- `nogoods: list[Nogood]` — recorded contradictions
- `repos: dict[str, str]` — name-to-path mapping for multi-repo tracking
- `log: list[dict]` — an append-only audit trail of every propagation event

### Core CRUD

**`add_node`** — Inserts a node, wires up the dependents index, computes initial truth value. A node with no justifications is a **premise** (IN by default). A node with justifications gets its truth computed immediately via `_compute_truth`. This is the only place new nodes enter the graph.

**`retract`** — Forces a node OUT regardless of its justifications. Sets `_retracted` in metadata (which `_propagate` checks to avoid overriding manual retractions). Returns the full list of IDs whose truth values changed, including cascade victims.

**`assert_node`** — The inverse of retract. Forces a node IN, clears the `_retracted` flag, and propagates restoration through dependents.

### Truth Computation

**`_compute_truth`** — Disjunctive: a node is IN if **any** justification is valid. For premises (no justifications), it returns the current value unchanged.

**`_justification_valid`** — The core TMS rule. An SL/CP justification is valid when:
1. All antecedents (inlist) are IN
2. All outlist nodes are OUT (or don't exist in the network)

This is what enables non-monotonic reasoning — "believe X unless Y is believed."

**`_propagate`** — BFS from a changed node through `dependents`. For each dependent, recomputes truth; if it changed, enqueues it. Skips nodes with `_retracted` metadata to respect manual retractions. This is the workhorse that makes cascades happen.

**`recompute_all`** — Iterates to fixpoint over all derived (non-retracted) nodes. Needed after bulk operations like import where incremental propagation wasn't run. Bounded by `len(nodes) + 1` iterations.

### Dependency-Directed Backtracking

**`add_nogood`** — Records a contradiction (a set of nodes that shouldn't all be IN simultaneously). If the contradiction is active, calls `find_culprits` to identify which premise to retract.

**`find_culprits`** — For each nogood node, traces backward via `trace_assumptions` to find all supporting premises. Then ranks them by **entrenchment** — premises with fewer dependents, no source backing, and speculative types (PREDICTED, DERIVED) are retracted first. Axioms and observations are protected.

**`_entrenchment`** — Scoring function: premises get +100, source-backed +50/+25, each dependent +10, plus type-based bonuses (AXIOM/WARNING: 90, OBSERVATION: 80, DERIVED: 40, etc.).

### Dialectical Operations

**`challenge`** — Creates a challenge node (a premise, so IN) and adds it to the target's outlist. Since the challenge is IN and now in the outlist, the target's justification becomes invalid and the target goes OUT. If the target was a premise, it gets converted to a justified node with an SL justification having the challenge in its outlist.

**`defend`** — Counter-challenges a challenge. Implemented as `self.challenge(challenge_id, ...)` — the defense is a premise in the challenge's outlist, so the challenge goes OUT, which restores the original target. This creates a three-layer stack: target ← challenge ← defense.

**`supersede`** — Adds the new node to the old node's outlist. When the new node is IN, the old one goes OUT. Reversible: retracting the new node restores the old one.

### Structural Operations

**`convert_to_premise`** — Strips all justifications from a node, making it a premise (IN by default). Cleans up the dependents index for the removed antecedents/outlist. Used when an import created a contextual dependency that isn't actually logical.

**`summarize`** — Creates a summary node with an SL justification over a group of nodes. The summary is IN only when all summarized nodes are IN. Used by the compact module to reduce token budget.

**`add_justification`** — Appends a justification to an existing node. Propagates access tags and recomputes truth. This is how derived beliefs get additional support.

### Diagnostic/Query

**`explain`** — Recursive trace of why a node is IN or OUT. For IN nodes, finds the valid justification and recurses into antecedents. For OUT nodes, reports which antecedents failed and which outlist nodes violated.

**`trace_assumptions`** — Walks backward through justification chains to find all premise nodes (leaf nodes with no justifications). These are the assumptions a conclusion rests on.

**`trace_access_tags`** — Backward walk collecting `access_tags` metadata from every node in the dependency subgraph.

**`get_belief_set`** — Returns all IN node IDs.

**`verify_dependents`** / **`_rebuild_dependents`** — Integrity checks and repair for the dependents reverse index.

## Patterns

1. **Incremental propagation with BFS** — `_propagate` uses a `deque` for breadth-first traversal. Every mutating operation (`retract`, `assert_node`, `add_node`, `challenge`, etc.) calls `_propagate` to push changes through the graph. This avoids recomputing the whole network on every change.

2. **Disjunctive justifications** — A node is IN if *any* justification is valid (`_compute_truth` short-circuits on the first valid one). This is the standard TMS semantics: multiple justifications provide alternative support.

3. **Outlist for non-monotonic reasoning** — The outlist mechanism is the key differentiator from a simple dependency graph. It enables "believe X unless Y" patterns, which power supersession, challenges, defenses, and GATE beliefs.

4. **Metadata as escape hatch** — Structured state like `_retracted`, `retract_reason`, `superseded_by`, `challenges`, `access_tags`, and `summarized_by` all live in the generic `metadata` dict. This keeps the `Node` dataclass stable while allowing features to layer on state.

5. **Audit logging** — Every mutation records a timestamped event in `self.log`. This is an append-only list used for debugging propagation behavior.

6. **Challenge/defend as self-similar recursion** — `defend` is implemented as `self.challenge(challenge_id, ...)`. The same outlist mechanism that makes challenges work also makes defenses work, creating an arbitrarily deep dialectical stack.

## Dependencies

**Imports:** Only stdlib (`collections.deque`, `datetime`) plus the package's own data types (`Node`, `Justification`, `Nogood` from `__init__.py`). Zero external dependencies — the network is a pure in-memory computation.

**Imported by:** Nearly everything. The `Network` class is used directly by `storage.py` (persistence), `api.py` (HTTP/CLI interface), `compact.py` (summarization), `check_stale.py` (staleness detection), `import_beliefs.py` / `import_agent.py` (bulk import), `export_markdown.py` (rendering), and 18+ test files. It is the gravitational center of the codebase.

## Flow

A typical lifecycle:

1. **Build phase** — `add_node` calls build the graph. Each node gets its dependents wired and initial truth computed.
2. **Mutation** — `retract`, `assert_node`, `challenge`, `defend`, `supersede`, or `add_justification` change a node's state.
3. **Propagation** — The mutation calls `_propagate`, which BFS-walks dependents, recomputing truth at each step and continuing only where values actually changed.
4. **Query** — `explain`, `trace_assumptions`, `get_belief_set` read the current state without modifying it.
5. **Contradiction handling** — `add_nogood` detects active contradictions and uses `find_culprits` + `retract` to resolve them automatically.

## Invariants

1. **Node IDs are unique** — `add_node` raises `ValueError` if the ID already exists.
2. **Dependents index mirrors justifications** — Every antecedent and outlist reference in a justification is reflected in the referenced node's `dependents` set. `verify_dependents` checks this. (Known issue: the index can drift if outlist nodes are added after the referencing justification.)
3. **`_retracted` nodes are sticky** — `_propagate` skips nodes with `_retracted` metadata, so automatic propagation won't override a manual retraction.
4. **Premises default to IN** — Any node with no justifications starts as IN. `_compute_truth` returns the current value for premises, so only explicit `retract` can take them OUT.
5. **Propagation terminates** — BFS with a `visited` set guarantees no infinite loops even in cyclic dependency graphs.

## Error Handling

Straightforward: `KeyError` for missing node IDs, `ValueError` for duplicate IDs. No try/except anywhere — errors propagate to callers. The `_propagate` method handles dangling dependents gracefully by logging a warning and skipping, rather than raising. This is important because the dependents index can contain stale references.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Defines the `Node`, `Justification`, and `Nogood` dataclasses that this module operates on
- [file] `reasons_lib/storage.py` — How the in-memory `Network` is serialized to/from SQLite and JSON; round-trip fidelity matters
- [function] `reasons_lib/check_stale.py:check_stale` — Uses the network to detect beliefs whose source code has changed, triggering re-evaluation
- [file] `tests/test_dialectical.py` — Tests for challenge/defend/supersede; the best way to understand the dialectical stack
- [general] `outlist-dependents-tracking` — The known issue where outlist nodes aren't always tracked in the dependents index, causing GATE beliefs to not auto-update

## Beliefs

- `network-is-pure-in-memory` — The `Network` class has zero I/O; all persistence is handled by `storage.py` and other modules that wrap it
- `node-truth-is-disjunctive` — A node is IN if *any* justification is valid; all justifications must be invalid for the node to be OUT
- `retracted-flag-blocks-propagation` — Nodes with `_retracted` in metadata are skipped by `_propagate`, preventing automatic truth restoration from overriding manual retractions
- `entrenchment-prefers-retracting-speculation` — `find_culprits` sorts by entrenchment ascending, so PREDICTED/DERIVED/unsourced nodes are retracted before AXIOM/OBSERVATION/sourced ones
- `challenge-converts-premises-to-justified` — When a premise is challenged, it gains an SL justification with an empty antecedent list and the challenge in its outlist, changing its truth-maintenance semantics permanently

