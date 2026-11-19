---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/network.py

# `reasons/network.py` — Dependency Network with Truth Maintenance

## Purpose

This module implements the **core data structure of an RMS (Reasoning/Reputation Management System)**: a Truth Maintenance System (TMS) following Doyle's 1979 algorithm. It owns the in-memory belief network that every other module operates on.

Responsibilities:
- Persist nodes, justifications, and nogoods in a live graph
- Compute each node's `IN`/`OUT` truth value from the SL/CP justifications
- Propagate changes through the graph on every mutation (retract, assert, supersede, challenge, defend, add/remove justification)
- Maintain a reverse `dependents` index for BFS propagation
- Resolve contradictions via dependency-directed backtracking (entrenchment-ranked culprits)
- Support dialectics (challenge/defend), supersession chains, summaries, and explanations
- Emit an audit trail (`self.log`) of every propagation event

It is deliberately **not** a persistence layer (see `storage.py`) or an LLM-facing API (`api.py`) — it is the pure semantic engine those layers wrap.

---

## Key Components

### `Network` attributes

| Attribute | Type | Purpose |
|---|---|---|
| `nodes` | `dict[str, Node]` | All nodes keyed by ID (retracted nodes stay here to enable restoration) |
| `nogoods` | `list[Nogood]` | Recorded contradictions; IDs like `nogood-a-b` with `-2`, `-3` suffixes |
| `repos` | `dict[str, str]` | `name → path` map for repo-backed nodes |
| `log` | `list[dict]` | Every propagation event with ISO timestamp |
| `meta` | `dict[str, str]` | Free-form schema metadata (schema_version, project_name) |

### Dependents index management

- **`_rebuild_dependents()`** — canonical rebuild; zeros every `dependents` set then adds `node.id` to each antecedent's and each outlist's dependents.
- **`verify_dependents() -> list[str]`** — returns error strings comparing the live index to what justifications imply (`extra` / `missing`). Empty list = consistent.

### Node lifecycle

- **`add_node(id, text, justifications=None, …) -> Node`**
  - Raises `ValueError` if `id` already exists.
  - Fills `created_at`/`updated_at` with `datetime.now(timezone.utc).isoformat(timespec="seconds")` if blank.
  - Registers the new node as a dependent of **every antecedent *and* every outlist node**.
  - Lazily computes `text_hash` and per-justification `content_hash` via `merkle`.
  - Calls `_inherit_access_tags`.
  - **If no justifications → truth_value = `"IN"` (premise); otherwise `_compute_truth(node)`.**

- **`retract(node_id, reason="") -> list[str]`** — force-OUT. Sets `_retracted` metadata and `retracted_at`, seeds `_propagate(node_id)`. Premise vs. derived makes no difference — retraction is always honored until reasserted.

- **`assert_node(node_id) -> list[str]`** — force-IN, clears `_retracted`, propagates. No-op returning `[]` if already IN.

- **`convert_to_premise(node_id)`** — strips all justifications, discards this node from antecedents'/outlist's `dependents`, forces IN, and propagates.

- **`remove_justification(node_id, index)`** — refuses to empty a derived node (`ValueError` suggesting `convert_to_premise` or `retract`), refreshes `dependents` only for refs no longer mentioned by any remaining justification.

- **`add_justification(node_id, justification)`** — registers dependents for the new justification, recomputes hash if missing, re-inherits access tags, recomputes truth, propagates if changed.

### Contradictions & backtracking

- **`_entrenchment(node_id) -> int`** — heuristic score; premises get `+100`, sourced `+50`, hashed `+25`, `10 × len(dependents)`, plus `beliefs_type` weights (`AXIOM/WARNING: 90`, `OBSERVATION: 80`, `DERIVED: 40`, `PREDICTED: 30`, `NOTE: 10`, default 20).
- **`find_culprits(nogood_node_ids) -> list[dict]`** — for every IN node, walks back to premises via `trace_assumptions`; returns candidate premises **sorted ascending by entrenchment** (least entrenched = safest to retract first) with `would_resolve`, `dependent_count`, `entrenchment`.
- **`add_nogood(node_ids) -> list[str]`** — deduplicates IDs to form `nogood-<sorted-joined>`, suffixing `-2`, `-3` if taken; records `Nogood` with `discovered` timestamp; **short-circuits if not all nodes IN**; retracts `culprits[0]` (or the fewest-dependent nogood node as fallback).

### Dialectics

- **`supersede(old_id, new_id, transitive=False) -> dict`**
  - Appends `new_id` to `old_id`'s outlist on every justification, or synthesizes `Justification(type="SL", antecedents=[], outlist=[new_id])` if the old node was a premise.
  - With `transitive=True`, walks the `supersedes` metadata chain to also supersede ancestors.
  - Registers `new_id` in `old_id.dependents` **in addition to** the standard outlist dependents logic (`self.nodes[new_id].dependents.add(target_id)`).
  - Returns `{"old_id", "new_id", "changed"}`.

- **`challenge(target_id, reason, challenge_id=None) -> dict`**
  - Creates a premise node (IN) with `challenge_target` metadata.
  - Adds the challenge ID to every existing justification's `outlist` **or**, for premises, converts them to a justified node whose outlist contains the challenge.
  - Registers `challenge_node.dependents.add(target_id)`.
  - Appends to `target.metadata["challenges"]`.

- **`defend(target_id, challenge_id, reason, defense_id=None) -> dict`**
  - Delegates to `challenge(challenge_id, …)` — defense is just a challenge against the challenge.
  - Adds `defense_target` and `defends` metadata.

### Propagation & truth evaluation

- **`_propagate(changed_id) -> list[str]`** — BFS from `changed_id` via `dependents`. Skips `_retracted` nodes and logs `"warn"` for **dangling dependents** (IDs referenced but no longer nodes). Recomputes `_compute_truth` per dependent and enqueues only on actual change.
- **`_compute_truth(node) -> str`** — iterates justifications **backwards**; first valid makes it IN and sets `supporting_justification = i`. Returns `"OUT"` if none valid. **For premises returns current truth_value** (keeps premise state).
- **`_justification_valid(j) -> bool`** — for `SL`/`CP`: all antecedents must be IN **and** all outlist must be OUT (missing outlist nodes count as OUT). Other types → `False`.
- **`recompute_all() -> list[str]`** — fixpoint loop up to `len(nodes) + 1` passes; only touches justified, non-retracted nodes; logs `"recompute"` events.
- **`explain(node_id, _visited=None, _path=None) -> list[dict]`** — recursive trace; detects cycles via `_path` (reports "circular dependency"); prefers `supporting_justification` but falls back to the last valid one; for OUT nodes it lists `failed_antecedents` and `violated_outlist`.

### Access tags

- **`_inherit_access_tags(node)`** — unions `access_tags` from **antecedents only** (not outlist) into `node.metadata["access_tags"]`, sorted.
- **`_propagate_access_tags(changed_id)`** — BFS forward via `dependents`, re-inheriting until fixpoint; prunes branches whose tags didn't change.
- **`trace_access_tags(node_id)`** — DFS backward through antecedents, unioning `access_tags` from the entire subgraph.

### Assumptions

- **`trace_assumptions(node_id) -> list[str]`** — DFS backward through antecedents; collects nodes with no justifications (premises). **Does not traverse outlist**; a premise that is only referenced as an outlist blocker is not returned.

### Reporting

- **`summarize(summary_id, text, over, source="")`** — creates an SL node with `antecedents=over`, label `"summarizes"`, metadata `{"summarizes": over}`; back-references each covered node in `summarized_by`.
- **`get_belief_set() -> list[str]`** — IDs where `truth_value == "IN"`.
- **`_log(action, target, value)`** — appends `{timestamp, action, target, value}` with `datetime.now().isoformat(timespec="seconds")`.

---

## Patterns

1. **Bellman-style BFS propagation** through a reverse `dependents` index. Every mutation calls `_propagate` to reach a fixpoint from a single seed; `recompute_all` provides a global fixpoint for import paths.
2. **Justification-based truth** (SL with outlist) rather than special-cased rules. Supersession, challenge, defense, and even "summarize" all reduce to outlist/inlist constraints.
3. **Metadata side-channel** (`_retracted`, `superseded_by`, `supersedes`, `challenges`, `summarized_by`, `access_tags`) — used for reporting and traversal-ordering, but not for truth computation (except `_retracted`).
4. **Lazy Merkle hashing** — `from .merkle import …` inside the function body to avoid a circular import at module load; hashing is idempotent (`if not …`).
5. **Reversible mutation** — retracted nodes remain in `self.nodes`; `_compute_truth` for premises returns the *current* value (so an explicitly retracted premise stays OUT even though it has no justifications).
6. **Dependency-directed backtracking** — `add_nogood` doesn't just pick any culprit; it ranks by entrenchment to protect evidence.
7. **Deterministic IDs** — nogoods (`nogood-<sorted>` with numeric suffix), challenges (`challenge-<target>[-n]`), defenses (`defense-<challenge>[-n]`).

---

## Dependencies

**Imports:**
- `collections.deque` — BFS frontier in `_propagate` and `_propagate_access_tags`.
- `datetime.datetime`, `datetime.timezone` — `retracted_at`, `created_at`, `updated_at`, log timestamps.
- `. (Node, Justification, Nogood)` — dataclass-like models from `reasons/__init__.py`.
- `.merkle` — `compute_text_hash`, `compute_content_hash`; imported **lazily inside methods** so module import doesn't require merkle.

**Imported by (via `reasons`):** `api.py`, `check_stale.py`, `compact.py`, `export_card.py`, `export_markdown.py`, `import_agent.py`, `import_beliefs.py`, `storage.py`. The `reasons_lib` sibling mirrors this module one-for-one, and ~25 tests exercise it (`test_network.py`, `test_backtracking.py`, `test_outlist.py`, `test_dialectical.py`, `test_merkle.py`, `test_dependents_integrity.py`, `test_dangling_dependents.py`, etc.).

---

## Flow

**Adding a derived node:**
```
add_node(id, text, justifications=[J])
  → register dependents for J.antecedents + J.outlist
  → insert self.nodes[id]
  → compute text_hash / content_hash (if missing)
  → _inherit_access_tags(node)
  → _compute_truth(node)          # walks justifications backward
  → _log("add", id, tv)
```

**Retraction cascade:**
```
retract(id, reason)
  → set truth_value="OUT", metadata["_retracted"]=True, retracted_at
  → _log("retract", id, reason)
  → _propagate(id)                 # BFS over self.nodes[id].dependents
       for each dependent d:
         if d._retracted: skip
         _compute_truth(d)         # any valid justification → IN
         if changed: update, enqueue
```

**Contradiction:**
```
add_nogood([a,b])
  → record Nogood with dedup'd id
  → if not all IN: return []
  → find_culprits([a,b])
       trace_assumptions per node → premises
       score each premise by _entrenchment
       sort ascending
  → retract(culprits[0]["premise"])
```

**Supersession:**
```
supersede(old, new, transitive=False)
  → for each target in {old} ∪ (transitive ancestors):
       append new to every justification's outlist
         (or synthesize SL justification if it was a premise)
       dependents[new].add(target)
       metadata["superseded_by"] = new
       recompute; propagate on change
  → new.metadata["supersedes"].append(old)
```

---

## Invariants

- **Node IDs are unique** — `add_node` raises `ValueError` if `id in self.nodes`.
- **`dependents` is derived, not authoritative** — `verify_dependents()` is the source of truth; every mutation must update it (add_node, add_justification, remove_justification, convert_to_premise, supersede, challenge).
- **Outlist nodes are dependents** — a change to an outlist node's truth re-evaluates the dependent (necessary for reversible supersession/defense).
- **Premises default IN** — no justifications → `truth_value = "IN"` at creation. `_compute_truth` is a no-op for premises (returns current value).
- **A node is derived-IN if *any* justification is valid** — `_compute_truth` returns IN at the first valid justification, so removing one justification may not change the truth value.
- **`_justification_valid` requires all antecedents IN *and* all outlist OUT** — AND semantics, non-monotonic.
- **Retracted nodes stay in the graph** — this is what enables restoration without rederivation. `_propagate` explicitly skips them (`if dep.metadata.get("_retracted"): continue`).
- **A derived node must retain ≥1 justification** — `remove_justification` refuses on the last one; the correct escape hatch is `convert_to_premise`.
- **Nogood IDs are deterministic** — sorted-and-joined node IDs, suffixed `-N` on collision.
- **`transitive=True` on `supersede` walks the `supersedes` chain** — only the newest version in a chain stays IN.
- **`_propagate_access_tags` converges** — branches only recurse when `old_tags != new_tags`.
- **`explain` handles cycles** — `_path` set detects revisits and emits a "circular dependency" step.
- **`recompute_all` is bounded** — `max_iterations = len(self.nodes) + 1`, so worst case is O(n²) but guaranteed to terminate (fixpoint).

---

## Error Handling

| Situation | Behavior |
|---|---|
| Missing node | `KeyError(f"Node '{id}' not found")` — raised from all public mutators |
| Duplicate node | `ValueError(f"Node '{id}' already exists")` from `add_node`, `summarize`, `challenge`, `defend` |
| Removing last justification | `ValueError` pointing at `convert_to_premise` / `retract` |
| Justification index out of range | `IndexError` from `remove_justification` |
| Dangling dependent | Logged as `{"action": "warn", …}` inside `_propagate`; skipped (no exception) |
| Nogood node IDs not all IN | Silent short-circuit returning `[]` — the contradiction is recorded but not active |
| Circular justification in `explain` | Returns a single step with `reason="circular dependency"` |
| Premise `_compute_truth` | Returns `node.truth_value` unchanged (intentional — no way to "compute" a premise) |
| Unknown justification type | `_justification_valid` returns `False` (so an unsupported type makes the node OUT) |

Retraction and assertion are idempotent — a no-op returns `[]` rather than raising.

---

## Topics to Explore

- [file] `reasons/__init__.py` — Defines `Node`, `Justification`, and `Nogood`, the exact shapes this module mutates (`supporting_justification`, `text_hash`, `outlist`, `.label`).
- [file] `reasons/merkle.py` — How `compute_text_hash` and `compute_content_hash` turn nodes/justifications into content-addressable hashes; critical for `content_hash` invariants.
- [function] `reasons/network.py:_entrenchment` — The heuristic that decides which premise gets retracted in `add_nogood`; drives user-visible backtracking semantics.
- [file] `tests/test_outlist.py` — Canonical examples of the SL-outlist mechanism: `supersede`, `challenge`, `defend`, and restoration.
- [file] `tests/test_backtracking.py` — Verified behavior of `find_culprits` and `add_nogood` ordering (least-entrenched first, premise protection).
- [general] `tms-doyle-semantics` — Review Doyle 1979 SL/CP semantics to sanity-check `_justification_valid` and whether CP should differ from SL beyond the current no-op.

## Beliefs

- `dependents-index-is-derived-from-justifications` — The `dependents` reverse index is reconstructible via `_rebuild_dependents()` and validated by `verify_dependents()`; no independent source of truth exists.
- `outlist-nodes-are-dependents` — Unlike pure antecedent-only TMSes, this implementation registers outlist nodes in `dependents`, which is what lets retracting a superseder restore the superseded node.
- `premises-default-in-and-skip-compute` — `add_node` sets `truth_value = "IN"` for zero-justification nodes, and `_compute_truth` returns the current value for premises rather than recomputing.
- `node-is-in-if-any-justification-valid` — `_compute_truth` returns IN at the first valid justification scanned in reverse order and sets `supporting_justification` to that index.
- `retracted-nodes-are-skipped-during-propagation` — `_propagate` short-circuits any dependent with `metadata["_retracted"] == True` so explicitly retracted nodes are not silently reasserted by propagation.
- `nogood-short-circuits-when-not-all-nodes-in` — `add_nogood` records the `Nogood` object but returns `[]` and performs no backtracking unless every listed node is currently `IN`.
- `supersede-is-reversible-via-outlist` — Because `supersede` uses outlist entries rather than direct truth-value mutation, retracting the `new_id` restores the `old_id` through normal propagation.
- `removing-last-justification-is-rejected` — `remove_justification` raises `ValueError` on the final justification; callers must use `convert_to_premise` or `retract` to change a derived node into a premise.
