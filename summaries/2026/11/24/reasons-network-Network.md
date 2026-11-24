---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: Network in reasons/network.py

# `Network` — Dependency Network (TMS Core)

## Purpose

`Network` is the core truth-maintenance (TMS) data structure of the RMS. It stores a directed graph of `Node`s (claims / beliefs) connected by `Justification`s, and it is responsible for:

- **State**: holding every node, its justifications, its `truth_value` (`IN` / `OUT`), and a reverse `dependents` index.
- **Reasoning**: implementing Doyle's (1979) TMS semantics — a node is `IN` iff at least one of its justifications is currently valid.
- **Propagation**: cascading truth-value changes through the dependency graph when a node is retracted, asserted, or re-justified.
- **Non-monotonicity**: honoring `outlist` (defeaters) in `SL`/`CP` justifications ("believe X *unless* Y").
- **Backtracking**: resolving `Nogood`s (contradictions) by retracting the least-entrenched premise responsible.
- **Dialectics**: `challenge` / `defend` machinery built on the same outlist mechanism.
- **Auditability**: capturing every propagation event in `self.log`.
- **Access control**: inheriting/forwarding `access_tags` metadata and tracing tags over a subgraph.

It is intentionally decoupled from persistence: construction, import, and loading all funnel through these methods.

---

## Contract

### Invariants

After all propagation has settled (i.e. no `_propagate` queue remains):

1. **Truth consistency for derived nodes** — `node.truth_value == self._compute_truth(node)` for every node with `justifications` and no `_retracted` metadata. `IN` ⟺ at least one valid justification exists; `supporting_justification` is set to the index of the newest valid one.
2. **Truth stickiness for premises** — a node with no `justifications` keeps whatever `truth_value` it currently has; `_compute_truth` never overrides it. `add_node` initializes to `IN`; `retract` sets `OUT`; `assert_node` resets to `IN`.
3. **Dependents index is derived** — `node.dependents` contains exactly the node IDs whose justifications reference `node.id` in `antecedents` *or* `outlist`. `_rebuild_dependents` reconstructs it; `verify_dependents` validates it.
4. **Retraction is sticky** — a node with `metadata["_retracted"] == True` is *skipped* by `_propagate` and `recompute_all`. It will not auto-restore even if a valid justification appears; only `assert_node` clears `_retracted`.
5. **Nogoods are immutable records** — appended to `self.nogoods`; they are not nodes and are not propagated to.
6. **Log is append-only** — `self.log` entries are never rewritten.
7. **Unknown references are tolerated** — justification antecedents/outlists may name IDs that don't exist in `self.nodes`; `_justification_valid` treats missing antecedents as `OUT` (invalidating the justification) and missing outlist entries as `OUT` (not blocking).

### Per-method preconditions / postconditions (selected)

| Method | Precondition | Postcondition |
|---|---|---|
| `add_node(id, ...)` | `id` not already in `self.nodes` | Node registered, hashes computed, `truth_value` set, dependents of antecedents/outlists updated, `add` logged |
| `retract(node_id)` | node exists | `truth_value == "OUT"`, `_retracted=True`, `retracted_at` set, cascade propagated; **idempotent** when already `OUT` (returns `[]` but still stamps `_retracted`) |
| `assert_node(node_id)` | node exists | `truth_value == "IN"`, `_retracted` cleared, `retracted_at=""`, cascade propagated; **returns `[]`** if already `IN` |
| `add_justification(node_id, j)` | node exists | `j` appended, dependents links added, tags inherited/forwarded, node recomputed & propagated |
| `challenge(target_id, reason)` | target exists; new challenge ID unused | Challenge node (premise) created, registered in target's outlist, target recomputed; target usually goes `OUT` |
| `defend(target_id, challenge_id, ...)` | target and challenge exist | A defense node (challenge-of-the-challenge) created; challenge typically goes `OUT`, restoring target |
| `convert_to_premise(node_id)` | node exists | `justifications=[]`, dependent links discarded, `truth_value` forced to `IN` |
| `remove_justification(node_id, index)` | node exists, has ≥2 justifications, index in range | Justification popped, orphaned dependent links pruned, node recomputed |
| `supersede(old_id, new_id, transitive=False)` | both exist | `new_id` appended to each target's outlists; target metadata `superseded_by` set; `new_id` metadata `supersedes` list updated |
| `add_nogood(node_ids)` | all IDs exist | New `Nogood` recorded; if all referenced nodes are `IN`, a culprit is retracted and the cascade returned |

---

## Parameters

Because this is a class, parameters span several methods. Grouped by purpose:

### Lifecycle & mutation
- **`id`, `text`** (`add_node`): Required. `text` feeds `compute_text_hash` and downstream `compute_content_hash`; empty strings are permitted but hash trivially.
- **`justifications: list[Justification] | None`** — `None` ⟺ premise (IN by default). Empty list also means premise. Non-empty triggers `_compute_truth`.
- **`metadata: dict | None`** — Arbitrary; keys with special meaning include `access_tags`, `beliefs_type` (used by `_entrenchment`), `_retracted`, `retract_reason`, `supersedes`, `summarized_by`, `summarizes`, `challenges`, `challenge_target`, `defense_target`, `defends`, `superseded_by`.
- **`created_at`, `updated_at`, etc.** — If `created_at` is empty, both `created_at` and `updated_at` are stamped with `datetime.now(timezone.utc)`; otherwise callers can backdate imports.

### Modification
- **`node_id: str`** — Every mutator hashes into `self.nodes`; missing keys raise `KeyError`.
- **`index: int`** (`remove_justification`) — Must satisfy `0 <= index < len(node.justifications)`.
- **`reason: str`** — Free text; stored on the node / nogood / log; not validated.

### Dialectics
- **`target_id`, `challenge_id`, `defense_id`** — `challenge_id` / `defense_id` are optional; when omitted the code derives `challenge-<target>` / `defense-<challenge>` and appends a numeric suffix on collision.
- **`reason`** — Becomes the challenge/defense node's `text`. Not hashed by `add_node` (see side effects).

### Backtracking
- **`node_ids: list[str]`** (`add_nogood`) — Must all exist; sorted to form the nogood slug/ID. Duplicates are technically allowed but produce the same slug.
- **`nogood_node_ids: list[str]`** (`find_culprits`) — Missing nodes are silently skipped; `OUT` nodes are ignored.

### Query-only
- **`node_id`, `_visited`, `_path`** (`explain`) — `_visited` and `_path` are internal recursion guards; callers should pass `None` and let recursion manage them.

---

## Return Value

Return shapes cluster into three patterns:

1. **`list[str]` of changed node IDs** — `retract`, `assert_node`, `recompute_all`, `add_nogood`. The returned list *includes* the directly affected node; downstream cascades are appended. Order is BFS order for propagation.
2. **`dict` with `changed`** — `supersede`, `add_justification`, `challenge`, `defend`, `convert_to_premise`, `remove_justification`, `summarize`. Each also carries method-specific fields (`old_id`/`new_id`, `old_truth_value`/`new_truth_value`, `removed`, `remaining`, `summary_id`, etc.).
3. **`list[dict]`** — `explain` returns step records; `find_culprits` returns `[{"premise", "would_resolve", "dependent_count", "entrenchment"}]` sorted ascending by entrenchment.

`verify_dependents` returns `[]` on success, one string per discrepancy otherwise.

Callers **must** treat the `changed` list as the authoritative set of nodes whose `truth_value` shifted; mutation methods do *not* re-raise or throw when propagation is a no-op.

---

## Algorithm

### 1. Truth evaluation — `_compute_truth` / `_justification_valid`

```
_compute_truth(node):
    if not node.justifications: return node.truth_value   # premise: sticky
    for i in reverse(range(len(justifications))):
        if _justification_valid(justifications[i]):
            node.supporting_justification = i
            return "IN"
    node.supporting_justification = None
    return "OUT"

_justification_valid(j):        # only types SL, CP are recognized
    inlist_ok  = all(a exists and nodes[a].truth_value == "IN" for a in j.antecedents)
    outlist_ok = all(o does not exist or nodes[o].truth_value == "OUT" for o in j.outlist)
    return inlist_ok and outlist_ok
```

Key consequences:
- **Missing antecedents invalidate** (they can't be `IN`); **missing outlist entries do not block** (absent ⟹ treated `OUT`).
- Justifications are examined **newest-first**, which makes the most recently added justification the designated support (`supporting_justification` points to the last index that wins).

### 2. Propagation — `_propagate`

BFS over `node.dependents` starting at the changed node:

```
queue = deque([changed_id]); visited = {changed_id}
while queue:
    current = popleft()
    for dep_id in current.dependents:
        if dep_id in visited:            continue
        if dep_id not in nodes:          log warn; continue   # dangling
        dep = nodes[dep_id]
        if dep.metadata.get("_retracted"): continue           # sticky skip
        old = dep.truth_value; new = _compute_truth(dep)
        if old != new:
            dep.truth_value = new
            changed.append(dep_id); visited.add(dep_id); queue.append(dep_id)
```

Because `dependents` is populated from *both* `antecedents` and `outlist`, a change to a defeater triggers the same propagation as a change to a supporting antecedent. The `visited` set guarantees cycle safety. Retracted nodes are treated as frozen (see invariant 4).

### 3. Dependency index maintenance

- **Incremental**: `add_node`, `add_justification`, `supersede`, `challenge` all do `self.nodes[target_id].dependents.add(source_id)`-style updates.
- **Decrement**: `convert_to_premise` and `remove_justification` `discard` links that no longer appear in any remaining justification.
- **Rebuild**: `_rebuild_dependents` wipes and re-derives everything from justifications. `verify_dependents` re-derives a shadow index and diffs against live state, returning human-readable diffs.

### 4. Backtracking — `find_culprits` → `add_nogood`

```
find_culprits(nogood_node_ids):
    for each IN node in nogood_node_ids:
        assumptions = trace_assumptions(node)          # DFS to premises
        collect premise → would_resolve mapping
    for each premise:
        entrenchment = _entrenchment(premise)
        candidate = {premise, would_resolve, dependent_count, entrenchment}
    sort candidates by entrenchment ascending
    return candidates
```

`add_nogood` then picks `culprits[0]["premise"]` — i.e. **the least-entrenched premise** — and calls `retract` on it. If no culprits exist it falls back to the nogood node with the fewest dependents. Because `_entrenchment` adds `100` for premise status, `50` for `source`, `25` for `source_hash`, `10 * |dependents|`, plus a `beliefs_type` bonus, "least entrenched" tends to favor unsourced, LEAF-like, low-type-weight premises. **Note: the docstring says "fewest dependents", but the actual sort key is entrenchment.**

### 5. Retraction vs. defeat — two distinct OUThes

- **`retract` / `add_nogood`** → hard, sticky `OUT`. `_retracted` marker prevents propagation from restoring it. Requires explicit `assert_node`.
- **Outlist defeat** → soft, auto-reversible. When a defeater goes `IN`, `_justification_valid` becomes `False`, and the cascade via `_propagate` sets the dependent `OUT`. If the defeater goes back `OUT`, the same cascade can restore it.

### 6. Cycle handling

- `_propagate` uses BFS + `visited`.
- `trace_assumptions`, `trace_access_tags`, `explain` all recurse with `visited` (and `explain` additionally tracks `_path` to detect and report `"circular dependency"`).
- `recompute_all` iterates to a fixpoint with cap `len(nodes) + 1`, which is sufficient for any acyclic graph regardless of insertion order and a safety net for cycles.

---

## Side Effects

- **In-place mutation** of `self.nodes`, `self.nogoods`, `self.log`, and each node's `truth_value`, `justifications`, `dependents`, `supporting_justification`, `metadata`, `text_hash`, `content_hash`, `created_at`/`updated_at`/`retracted_at`.
- **Justification outlist mutation** in `supersede`, `challenge`, `defend` — the list is *appended*, not replaced, so repeated calls are guarded by `if new_id not in j.outlist`.
- **`self.log` append** via `_log` for every auditable action (`add`, `retract`, `assert`, `propagate`, `add-justification`, `challenge`, `supersede`, `nogood`, `backtrack`, `convert-to-premise`, `remove-justification`, `recompute`, `warn`).
- **`metadata` writes** for `_retracted`, `retract_reason`, `supersedes`, `superseded_by`, `challenges`, `challenge_target`, `defense_target`, `defends`, `summarizes`, `summarized_by`, `access_tags`.
- **`recompute_all`** mutates `truth_value` without propagating to dependents in the same step — it relies on repeat passes to reach a fixpoint.
- **`challenge`** creates a `Node` **directly**, bypassing `add_node`. This means challenge nodes get **no Merkle hashes, no timestamps, no inherited `access_tags`**, and the dependent-maintenance side effects of `add_node` are re-implemented inline (only `challenge_node.dependents.add(target_id)`).

### Timezone inconsistency
`_log` and `add_nogood` use `datetime.now()` (naive local); `add_node`, `retract`, `assert_node` use `datetime.now(timezone.utc)`. Audit trails can therefore mix naive and aware timestamps.

---

## Error Handling

| Exception | Where |
|---|---|
| `KeyError("Node '<id>' not found")` | Every mutator/precondition (`retract`, `assert_node`, `add_justification`, `challenge`, `defend`, `convert_to_premise`, `remove_justification`, `summarize`, `add_nogood`, `explain`, `trace_*`, `find_culprits`) |
| `ValueError("Node '<id>' already exists")` | `add_node`, `summarize`, `challenge`, `defend` |
| `ValueError("Node '<id>' is a premise")` | `remove_justification` |
| `ValueError("only one justification")` | `remove_justification` (caller must use `convert_to_premise` or `retract`) |
| `IndexError` | `remove_justification` with out-of-range index |
| Swallowed / logged only | Dangling dependents in `_propagate` (`warn` action); missing nodes in `find_culprits` (silently skipped) |

`_justification_valid` **never raises** for unknown IDs — it treats absence as `OUT` for antecedents and as non-blocking for outlist. This is a deliberate design: networks may be loaded with dangling references, and validity degrades gracefully.

---

## Usage Patterns

1. **Build a belief graph**
   ```python
   net = Network()
   net.add_node("A", "All men are mortal", metadata={"beliefs_type": "AXIOM"})
   net.add_node("B", "Socrates is a man", source="plato")
   net.add_node("C", "Socrates is mortal",
                justifications=[Justification(type="SL", antecedents=["A", "B"])])
   ```
   `C` is `IN` because both antecedents are `IN` and the SL outlist is empty.

2. **Non-monotonic defeat**
   ```python
   net.add_node("C", ..., justifications=[
       Justification(type="SL", antecedents=["A", "B"], outlist=["wounded"]),
   ])
   net.add_node("wounded", "Socrates is wounded")   # IN by default
   ```
   `wounded` → `C` is `OUT`. Retracting/asserting `wounded` toggles `C` via BFS.

3. **Resolve a contradiction**
   ```python
   changed = net.add_nogood(["C1", "C2"])
   ```
   Adds a `Nogood`, retracts the least-entrenched premise underpinning `C1`/`C2`, returns full cascade.
   *Caller obligation*: inspect `changed` to know which assertions were invalidated.

4. **Dialectics**
   ```python
   r1 = net.challenge("C", "C is wrong because X", challenge_id=None)
   r2 = net.defend("C", r1["challenge_id"], "X is invalid (source Y)")
   ```
   Each call returns a `changed` list; `defend` restores `C` only if the challenge actually flipped `OUT` as expected.

5. **Supersede versions**
   ```python
   net.supersede("v1", "v2", transitive=True)
   ```
   `v2` becomes the `outlist` entry on `v1` (and any ancestors). `v1` goes `OUT` iff `v2` is `IN`; retract `v2` and `v1` returns naturally.

6. **Stabilize after bulk load**
   ```python
   net._rebuild_dependents()   # if importing without side effects
   changed = net.recompute_all()
   errors  = net.verify_dependents()   # expect []
   ```
   `recompute_all` iterates to a fixpoint, accommodating arbitrary insertion order.

---

## Dependencies

| Symbol | Where used | Note |
|---|---|---|
| `from . import Node, Justification, Nogood` | Everywhere | Package `reasons/__init__` exports the data classes |
| `collections.deque` | `_propagate`, `_propagate_access_tags` | BFS queues |
| `datetime.now`, `timezone` | timestamp stamping | Mixed UTC / naive (see side effects) |
| `reasons.merkle.compute_text_hash` / `compute_content_hash` | `add_node`, `add_justification` | **Lazy-imported** inside methods to avoid eager circular imports |
| `Justification(type="SL", ...)` | `supersede`, `challenge` | Auto-injects SL justifications when converting premises |

---

## Topics to Explore

- [file] `reasons/merkle.py` — Content-address hashing used by `add_node` and `add_justification`; explains why `content_hash` is lazily computed and how stale detection works.
- [function] `reasons/network.py:_justification_valid` — The single locus of SL/CP semantics; every propagation decision reduces to this predicate.
- [general] `truth-maintenance-system` — Doyle's original TMS formulation and how this implementation maps into it (sticky retraction vs. pure TMS recomputation is the notable divergence).
- [file] `tests/test_backtracking.py` — Concretely exercises `add_nogood` / `find_culprits` entrenchment ordering.
- [function] `reasons/network.py:_rebuild_dependents` — The canonical reverse-index reconstruction that `verify_dependents` must always agree with; find its call sites in storage/import code.

---

## Beliefs

- `network-computes-truth-from-justifications` — A derived node's `truth_value` is `IN` iff at least one `SL`/`CP` justification has all antecedents `IN` and all outlist entries `OUT`; premises bypass this and retain their current value.
- `retracted-nodes-are-sticky-out` — `retract` sets `_retracted` metadata, and both `_propagate` and `recompute_all` skip such nodes, so only `assert_node` can flip them back to `IN` — automated propagation will not.
- `challenge-bypasses-add-node` — `challenge` constructs `Node` directly instead of calling `add_node`, so challenge/defense nodes have no Merkle hashes, no timestamps, and no inherited `access_tags`.
- `add-nogood-retracts-least-entrenched-culprit` — Despite the docstring's "fewest dependents" phrasing, `add_nogood` retracts `culprits[0]`, which is sorted by `_entrenchment` ascending (dependent count is only one of several weighted factors).
- `missing-antecedent-invalidates-justification` — `_justification_valid` treats an antecedent ID that is not in `self.nodes` as non-`IN`, invalidating the justification rather than raising; missing outlist entries are similarly tolerated (treated as `OUT`).
- `dependents-index-is-reconstructible` — `node.dependents` is derivable purely from each node's `justifications` via `_rebuild_dependents`, and `verify_dependents` re-derives it to detect and report drift.
