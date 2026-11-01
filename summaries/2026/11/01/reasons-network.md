---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/network.py

# `reasons/network.py` — Dependency Network with Truth Value Propagation

## 1. Purpose

This file is the **core engine** of the entire project. It implements Doyle's (1979) Truth Maintenance System (TMS) as a dependency network where beliefs (nodes) are supported by justifications, and truth values propagate automatically through the graph.

Think of it this way: the project is a "reasoning engine" for managing a knowledge base of beliefs. `network.py` is the single class (`Network`) that owns:

- The **graph structure** (nodes, edges via justifications, dependents index)
- **Truth value computation** (IN/OUT based on justification validity)
- **Cascading propagation** (when one node changes, all dependents recompute)
- **Contradiction resolution** (nogoods + dependency-directed backtracking)
- **Dialectical operations** (challenge, defend, supersede)

Every other module in the project (`api.py`, `storage.py`, `import_beliefs.py`, `check_stale.py`, etc.) delegates to this class for its core reasoning logic. It's the single source of truth for the belief network.

## 2. Key Components

### `Network` class

The sole class in the file. Its state:

| Field | Type | Purpose |
|-------|------|---------|
| `nodes` | `dict[str, Node]` | All nodes in the network, keyed by ID |
| `nogoods` | `list[Nogood]` | Recorded contradictions |
| `repos` | `dict[str, str]` | Name→path mapping for repos |
| `log` | `list[dict]` | Audit trail of propagation events |
| `meta` | `dict[str, str]` | Schema version, project name, etc. |

### Core Truth Propagation

- **`_compute_truth(node) -> str`** — Returns `"IN"` or `"OUT"`. A node is IN if **any** justification is valid. Prefers the newest valid justification (last-added) as the `supporting_justification`. Nodes with no justifications are premises (keep their current value).

- **`_justification_valid(j) -> bool`** — The semantic core. For both SL and CP types:
  - **Inlist (antecedents)**: ALL must be IN
  - **Outlist**: ALL must be OUT (or not exist)
  
  This is what enables **non-monotonic reasoning**: "believe X unless Y" is expressed as `antecedents=[X], outlist=[Y]`.

- **`_propagate(changed_id) -> list[str]`** — BFS through the `dependents` reverse index. For each dependent, recomputes truth value; if it changed, enqueues its dependents. Skips nodes flagged `_retracted` in metadata. Returns all node IDs whose truth value flipped.

### Dependency Index

- **`_rebuild_dependents()`** — The **single canonical** rebuild. Wipes all `node.dependents` sets, then walks every justification's `antecedents` and `outlist` to reconstruct the reverse edges. Comment explicitly says "All other rebuild call sites should delegate here."

- **`verify_dependents() -> list[str]`** — Independent re-derivation of the expected dependents index, then diffs against the live index. Returns human-readable error strings. This is the integrity checker (tested in `test_dependents_integrity.py`).

### Node Lifecycle

| Method | Effect |
|--------|--------|
| `add_node(...)` | Insert node, register dependents, compute Merkle hashes, inherit access tags, compute initial truth value |
| `retract(node_id, reason)` | Force OUT, cascade to dependents |
| `assert_node(node_id)` | Force IN, cascade to dependents |
| `convert_to_premise(node_id)` | Strip all justifications → node becomes IN-by-default |
| `add_justification(node_id, j)` | Append a justification, recompute, propagate |
| `remove_justification(node_id, index)` | Pop one justification, clean up orphaned dependents, propagate |

### Contradiction Resolution

- **`add_nogood(node_ids) -> list[str]`** — Records a contradiction (set of nodes that can't all be IN simultaneously). If all are currently IN, invokes `find_culprits()` to identify which premise to retract. Falls back to retracting the nogood node with fewest dependents. Returns all changed node IDs.

- **`find_culprits(nogood_node_ids) -> list[dict]`** — For each nogood node, traces back to its premises via `trace_assumptions()`. Scores each premise by `_entrenchment()`. Returns candidates sorted **least entrenched first**.

- **`_entrenchment(node_id) -> int`** — Heuristic scoring:
  - Premise (no justifications): +100
  - Has `source`: +50, has `source_hash`: +25
  - Per dependent: +10
  - `beliefs_type` metadata: AXIOM/WARNING=90, OBSERVATION=80, DERIVED=40, PREDICTED=30, NOTE=10, default=20

### Dialectical Operations

- **`challenge(target_id, reason, challenge_id?)`** — Creates a new premise node (IN by default) and adds it to the target's outlist. Target goes OUT. If target was a premise, it's converted to a justified node with an SL justification.

- **`defend(target_id, challenge_id, reason, defense_id?)`** — Calls `challenge()` on the *challenge node* itself (challenging the challenge). The defense is IN → challenge goes OUT → target is restored.

- **`supersede(old_id, new_id, transitive=False)`** — Adds `new_id` to `old_id`'s outlist. When `new_id` is IN, `old_id` goes OUT. **Reversible**: if `new_id` is later retracted, `old_id` comes back. With `transitive=True`, walks the `supersedes` metadata chain.

### Tracing & Explanation

- **`trace_assumptions(node_id) -> list[str]`** — DFS backward through justification antecedents, collecting all premise node IDs.
- **`trace_access_tags(node_id) -> list[str]`** — DFS backward collecting the union of `access_tags` from all nodes in the dependency subgraph.
- **`explain(node_id) -> list[dict]`** — Recursive trace producing a human-readable chain of why a node is IN or OUT, including circular dependency detection.

### Access Tags (ABAC-style propagation)

- **`_inherit_access_tags(node)`** — Merges `access_tags` from all antecedent nodes into the current node.
- **`_propagate_access_tags(changed_id)`** — BFS forward: if a node's tags changed, push updates to dependents.
- **`trace_access_tags(node_id)`** — Full backward walk for the union of all tags in the subgraph.

### Miscellaneous

- **`recompute_all() -> list[str]`** — Iterates to fixpoint, recomputing all non-retracted derived nodes. Bounded by `len(nodes) + 1` iterations.
- **`summarize(summary_id, text, over)`** — Creates an SL node with all `over` nodes as antecedents. IN when all are IN.
- **`get_belief_set() -> list[str]`** — All currently-IN node IDs.
- **`_log(action, target, value)`** — Appends to `self.log` with timestamp.

## 3. Patterns

- **Truth Maintenance System (Doyle 1979)**: The entire design is a direct implementation of TMS. Justifications are the "rules," nodes are "beliefs," and propagation is the "inference engine."

- **Non-monotonic reasoning via outlists**: The `outlist` mechanism ("unless" clauses) is the key trick. A justification is valid only when all antecedents are IN *and* all outlist nodes are OUT. This makes the system defeasible — adding a new fact can *retract* a previously-true belief.

- **BFS propagation with visited set**: `_propagate` uses `collections.deque` for O(1) popleft, a `visited` set to prevent cycles, and only enqueues nodes whose truth value actually changed. This keeps the cascade minimal.

- **Entrenchment-based backtracking**: Rather than random or alphabetical retraction, `find_culprits` uses a weighted scoring heuristic to prefer retracting speculative assumptions over evidence. This is "dependency-directed backtracking" — you only retract what's *causally responsible* for the contradiction.

- **Challenge/defense as symmetric mechanism**: `defend` is literally `challenge` applied to the challenge. The codebase models dialectic as a uniform operation, not a special case.

- **Reverse index (dependents)**: Each node maintains a `dependents` set (who depends on me). This is the reverse of the forward edges (antecedents/outlist in justifications). This makes propagation O(dependents) rather than O(all nodes).

- **Audit trail via `_log`**: Every mutation appends to `self.log`. This is a simple in-memory event log, not a database. It's used for debugging and the `check_stale` workflow.

- **Merkle hash laziness**: `compute_text_hash` and `compute_content_hash` are only computed if not already set. This enables incremental updates without recomputing the entire hash tree.

## 4. Dependencies

### This file imports:
- `collections.deque` — BFS queue
- `datetime.datetime, timezone` — Timestamps for audit log, retraction timestamps
- `.Node`, `.Justification`, `.Nogood` — Core data classes (defined in `reasons/__init__.py` or `reasons_lib/__init__.py`)
- `.merkle.compute_text_hash`, `.merkle.compute_content_hash` — Content-addressing (lazy, inside methods)

### This file is imported by:
- **Core modules**: `api.py`, `storage.py`, `check_stale.py`, `compact.py`, `import_agent.py`, `import_beliefs.py`
- **Export modules**: `export_card.py`, `export_markdown.py`
- **Mirror copy**: `reasons_lib/network.py` (appears to be a duplicate/shim)
- **~20 test files** covering: network basics, access tags, backtracking, dangling dependents, integrity, dialectic, nogood IDs, outlists, pinning sources, roundtrip, summarize, etc.

The project is widely adopted internally — this is a load-bearing class.

## 5. Flow

### Adding a node (`add_node`):
```
1. Validate ID doesn't exist
2. Create Node with timestamps
3. Register as dependent of all antecedent/outlist nodes (forward→reverse edge)
4. Insert into self.nodes
5. Compute Merkle hashes (lazy)
6. Inherit access tags from antecedents
7. Compute truth value:
   - Has justifications → _compute_truth() → _justification_valid() for each
   - No justifications → premise, truth_value = "IN"
8. Log the event
```

### Retraction cascade (`retract` → `_propagate`):
```
1. Set node.truth_value = "OUT", mark _retracted
2. _propagate(node_id):
   a. BFS queue starts with [node_id]
   b. For each node popped, iterate its .dependents:
      - Skip if visited or missing
      - Skip if _retracted
      - Compute new truth value
      - If changed: update, log, enqueue dependents
   c. Repeat until queue empty
3. Return all changed node IDs
```

### Contradiction resolution (`add_nogood`):
```
1. Record the Nogood
2. If not all nodes are IN → no-op
3. find_culprits():
   a. For each nogood node → trace_assumptions() → collect premises
   b. For each premise → check which nogood nodes it supports
   c. Score by _entrenchment()
   d. Sort ascending (least entrenched first)
4. Retract the top candidate (or fallback: nogood node with fewest dependents)
5. Return changed IDs
```

### Challenge → Defense cycle:
```
challenge(target, "reason"):
  → create challenge_node (premise, IN)
  → add challenge_node to target.outlist
  → target._compute_truth() → OUT (outlist violated)
  → propagate

defend(target, challenge_id, "counter-reason"):
  → challenge(challenge_id, "counter-reason")  # challenge the challenge
  → challenge_node goes OUT
  → target's outlist no longer violated
  → target._compute_truth() → IN
  → propagate
```

## 6. Invariants

1. **Dependents index consistency**: If node B has a justification referencing node A (as antecedent or outlist), then A's `dependents` set MUST contain B's ID. `_rebuild_dependents()` is the canonical rebuild; `verify_dependents()` is the checker. Violations produce error strings, not exceptions.

2. **Premise semantics**: A node with zero justifications is a premise. Its truth value is NOT computed — it retains whatever value was set (IN by default, or OUT if explicitly retracted). `_compute_truth` returns `node.truth_value` unchanged for premises.

3. **At most one `supporting_justification`**: When a node is IN, exactly one justification index is recorded as the "designated support." When OUT, it's `None`. This is set in `_compute_truth` and used by `explain()`.

4. **Nogood ID uniqueness**: `add_nogood` generates deterministic IDs (`nogood-{sorted-slug}`) with numeric suffixes for collisions. Duplicate nogoods are prevented.

5. **`_retracted` flag bypasses propagation**: In `_propagate`, nodes with `metadata["_retracted"] = True` are skipped. This means a manually-retracted node won't be "restored" by a dependent's change — it stays OUT until explicitly `assert_node`d.

6. **Access tag monotonicity**: `_inherit_access_tags` only *adds* tags (set union). Tags are never removed by inheritance. `_propagate_access_tags` pushes changes forward but never removes.

7. **Recomputability**: `recompute_all()` must converge to the same fixpoint regardless of node insertion order (bounded by `len(nodes)+1` iterations).

## 7. Error Handling

The code uses a **raise-early** strategy with specific exception types:

| Scenario | Exception | Where |
|----------|-----------|-------|
| Duplicate node ID on add | `ValueError` | `add_node`, `challenge`, `defend`, `summarise` |
| Missing node reference | `KeyError` | `retract`, `assert_node`, `challenge`, `defend`, `convert_to_premise`, `remove_justification`, `add_nogood`, `trace_*`, `explain`, `supersede` |
| Index out of range | `IndexError` | `remove_justification` |
| Removing last justification | `ValueError` | `remove_justification` (directs user to `convert_to_premise` or `retract`) |
| Node is a premise but you try to remove justification | `ValueError` | `remove_justification` |

**Swallowed/handled internally:**
- Dangling dependents (node ID in `dependents` set but not in `self.nodes`): logged as `"warn"` in the audit trail, then skipped. No exception.
- Outlist nodes that don't exist: `_justification_valid` treats them as satisfied (`o not in self.nodes` → True). A missing outlist node doesn't invalidate a justification.
- Circular dependencies in `explain()`: detected via `_path` set, returns a "circular dependency" step instead of recursing forever.
- `_propagate` on a node not in `self.nodes`: logged as warning, skipped.

**Notably absent**: There's no try/except anywhere in this file. All error paths are explicit raises. The only "soft" failures are the dangling-node warnings logged to `self.log`.

---

## Topics to Explore

- [file] `reasons/__init__.py` — Defines `Node`, `Justification`, and `Nogood` data classes. Understanding their fields (especially `antecedents`, `outlist`, `dependents`, `truth_value`, `supporting_justification`) is prerequisite to reading this file effectively.
- [function] `reasons/network.py:_compute_truth` — The 10-line function that determines the entire semantics of the system. The "prefer newest valid justification" ordering and the premise pass-through behavior have subtle implications for `explain()` and `recompute_all()`.
- [file] `reasons/repair.py` — Likely consumes `find_culprits` and `retract` to implement the higher-level "repair a contradiction" workflow. Understanding how it orchestrates multiple `Network` calls will clarify the API contract.
- [file] `tests/test_dialectical.py` — Exercises the challenge/defend/supersede interplay. This is the most complex behavioral path in the network and the tests will make the non-monotonic outlist semantics concrete.
- [general] `merkle-hash-incremental-updates` — How `compute_content_hash(node, justification, self)` uses the network to build a content hash, and what happens when a node's text or justifications change after the hash is computed.

## Beliefs

- `network-implements-doyle-tms` — The `Network` class is a direct implementation of Doyle's 1979 TMS algorithm with SL/CP justifications, non-monotonic outlists, and dependency-directed backtracking for contradiction resolution.
- `outlist-enables-defeasibility` — A justification is valid only when all outlist nodes are OUT; this makes the reasoning system non-monotonic (adding a fact can retract a previously true belief).
- `entrenchment-scores-premises-over-derivations` — `_entrenchment` assigns +100 to premises (no justifications) and +40 to DERIVED nodes, ensuring `find_culprits` always prefers retracting speculative assumptions over evidence.
- `dependents-index-must-match-justifications` — The reverse `dependents` index on every node is a derived structure that must be consistent with the forward edges in justifications; `verify_dependents()` is the invariant checker and `_rebuild_dependents()` is the sole canonical rebuild path.
- `retract-sets-retracted-flag-blocks-propagation` — Once `metadata["_retracted"]` is set to True, `_propagate` skips that node entirely, preventing automatic restoration; only explicit `assert_node` can clear the flag and re-enable propagation.
