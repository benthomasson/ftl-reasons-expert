# Function: add_nogood in reasons_lib/network.py

**Date:** 2026-04-23
**Time:** 16:36

## `add_nogood` — Dependency-Directed Backtracking for Contradiction Resolution

### 1. Purpose

`add_nogood` records that a set of nodes are mutually contradictory — they cannot all be believed simultaneously. It then **automatically resolves** the contradiction by tracing backward through justification chains to find the least-entrenched premise responsible, and retracts it.

This implements the "nogood" mechanism from Doyle's Truth Maintenance System (TMS). In a TMS, a nogood is a set of beliefs that reality says can't all be true at once. Rather than letting the human decide what to retract, the system uses **dependency-directed backtracking**: it doesn't just retract one of the contradicting nodes arbitrarily — it traces back to the *assumptions* (premises) that led to the contradiction and retracts the one that causes the least collateral damage.

### 2. Contract

**Preconditions:**
- All node IDs in `node_ids` must exist in the network. No partial membership is allowed.
- The network's justification graph must be in a consistent state (truth values reflect current justifications).

**Postconditions:**
- A `Nogood` record is always appended to `self.nogoods`, even if no retraction occurs.
- If all named nodes were IN, exactly one node is retracted (either a traced-back premise or a fallback nogood member), and the retraction cascade propagates through all dependents.
- If any named node was already OUT, no retraction happens — the contradiction is already resolved.

**Invariant:** After `add_nogood` returns, at least one of the nogood nodes is OUT (assuming all were IN on entry).

### 3. Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `node_ids` | `list[str]` | The IDs of nodes that form the contradiction. Must contain at least the nodes that are mutually inconsistent. All must exist in `self.nodes`. |

**Edge cases:**
- A single-element list is technically accepted but semantically odd — it says "this node contradicts itself."
- Duplicate IDs in the list are not guarded against; they'd work but produce a misleading nogood record.
- An empty list would pass validation, create a nogood record, and return `[]` (since `all([])` is `True` in Python, it would proceed to `find_culprits` with an empty set — which returns no candidates — then hit the fallback with an empty `candidates` list, causing an `IndexError`).

### 4. Return Value

Returns `list[str]` — the IDs of all nodes whose truth value changed as a result of the retraction cascade. This includes:
- The retracted node itself
- Any dependents that flipped from IN to OUT (or OUT to IN via restoration)

Returns an **empty list** if the contradiction is not currently active (at least one nogood node is already OUT).

The caller should use this to update any external views (e.g., re-render beliefs, update a UI).

### 5. Algorithm

**Step 1 — Validate.** Iterate `node_ids` and raise `KeyError` if any node doesn't exist.

**Step 2 — Record.** Create a `Nogood` record with a sequential ID (`nogood-001`, `nogood-002`, ...) and append it to `self.nogoods`. This is permanent — the nogood is recorded regardless of whether action is needed.

**Step 3 — Check if active.** If any node in the nogood set is already OUT, the contradiction isn't live. Return `[]` immediately.

**Step 4 — Find culprits.** Call `find_culprits(node_ids)`, which:
  - For each IN nogood node, traces back through justification chains to find all premises (leaf nodes with no justifications).
  - For each premise found, records which nogood nodes depend on it.
  - Sorts by **entrenchment** (ascending) — speculative assumptions sort before evidence-backed observations.

**Step 5 — Choose a victim.**
  - **Primary path:** If `find_culprits` returns candidates, pick the first one (least entrenched premise). This is the "smart" path — it retracts an *assumption* rather than a *conclusion*.
  - **Fallback path:** If no culprits are found (possible when all nogood nodes are premises themselves with no justification chains to trace), fall back to retracting the nogood node with the fewest direct dependents.

**Step 6 — Retract.** Call `self.retract(victim_id)`, which sets the victim to OUT and BFS-propagates through all dependents. Return the list of changed nodes.

### 6. Side Effects

- **Mutates `self.nogoods`**: Appends a new `Nogood` record unconditionally.
- **Mutates `self.log`**: Logs the nogood event, and optionally a backtrack event.
- **Mutates node truth values**: The retracted node and its dependents may flip from IN to OUT (via `self.retract`).
- **Mutates node metadata**: `retract()` sets `_retracted: True` on the victim.
- **No I/O**: All changes are in-memory. Persistence (to `reasons.db` or `network.json`) is the caller's responsibility.

### 7. Error Handling

| Exception | When |
|-----------|------|
| `KeyError` | Any node ID in `node_ids` doesn't exist in `self.nodes`. Raised eagerly before recording the nogood. |

**Unhandled edge case:** An empty `node_ids` list will create a nogood record, pass the `all_in` check (since `all([])` is `True`), get no culprits, then crash with `IndexError` on the fallback's `candidates[0]` access.

**No error is raised** if the same nogood is recorded twice — duplicates are silently accepted.

### 8. Usage Patterns

Typical usage in the reasons CLI or API:

```python
# User discovers that beliefs A and B contradict each other
changed = network.add_nogood(["belief-A", "belief-B"])
# changed tells you which nodes flipped — persist and report
```

The caller is responsible for:
- Persisting the network after the call (save to DB/JSON).
- Reporting which nodes changed to the user.
- The nogood is **permanent** — there's no `remove_nogood`. Once recorded, the system remembers this combination is contradictory even if the nodes are later reasserted.

The nogood ID scheme (`nogood-{len+1}`) assumes nogoods are never deleted. If a nogood were removed from the list, subsequent IDs could collide.

### 9. Dependencies

| Dependency | Purpose |
|------------|---------|
| `Nogood` (from `reasons_lib`) | Data class for the contradiction record |
| `self.find_culprits()` | Traces justification chains to locate responsible premises, sorted by entrenchment |
| `self.retract()` | Performs the actual retraction with BFS cascade propagation |
| `self.trace_assumptions()` | Used internally by `find_culprits` to walk justification chains |
| `self._entrenchment()` | Scoring function that protects evidence over speculation |
| `datetime` | Timestamps for the nogood record and log entries |

---

## Topics to Explore

- [function] `reasons_lib/network.py:find_culprits` — The entrenchment-sorted backtracking logic that decides *which* premise to sacrifice; understanding the scoring in `_entrenchment` is key to predicting system behavior
- [function] `reasons_lib/network.py:retract` — The retraction cascade mechanism — what actually happens to dependents when a node goes OUT, and how `_retracted` metadata interacts with `_propagate`
- [file] `tests/test_backtracking.py` — Test cases that exercise the dependency-directed backtracking path, including edge cases around which premise gets chosen
- [function] `reasons_lib/network.py:_propagate` — The BFS engine that pushes truth-value changes through the dependency graph; understanding visited-set semantics matters for cycles
- [general] `doyle-tms-nogoods` — Doyle's 1979 "A Truth Maintenance System" paper, specifically Section 4 on dependency-directed backtracking, which this code implements

---

## Beliefs

- `add-nogood-always-records` — `add_nogood` appends a `Nogood` record unconditionally, even when the contradiction is not currently active (not all nodes IN)
- `add-nogood-retraction-prefers-least-entrenched` — The primary retraction path chooses the premise with the lowest entrenchment score (speculative assumptions before evidence), not simply the node with fewest dependents
- `add-nogood-fallback-uses-dependent-count` — When `find_culprits` returns no candidates (all nogood nodes are premises), the fallback retracts the nogood member with the fewest direct dependents
- `add-nogood-empty-list-crashes` — An empty `node_ids` list passes validation and the `all_in` check but causes an `IndexError` in the fallback path because `candidates` is empty
- `nogood-ids-assume-append-only` — Nogood IDs are derived from `len(self.nogoods) + 1`, so deleting a nogood from the list would cause ID collisions on the next call

