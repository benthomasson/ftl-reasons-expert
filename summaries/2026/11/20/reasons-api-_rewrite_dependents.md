---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _rewrite_dependents in reasons/api.py

## Explanation of `_rewrite_dependents`

### Purpose

`_rewrite_dependents` is a private helper used by the deduplication path (`deduplicate(auto=True)` and `apply_dedup_plan`) to **surgically re-point all justifications and reverse-index entries** of a soon-to-be-retracted duplicate node (`old_id`) at the canonical node that is being kept (`new_id`).

Without this fix-up, every belief that was justified by the duplicate would lose its support the moment the duplicate is retracted, even though a logically equivalent node (`new_id`) is still IN. This helper makes the retraction "invisible" to downstream beliefs — they keep the same truth value because their justifications were rewritten to cite the surviving canonical node.

It exists as a separate function rather than inline logic because two callers (`deduplicate` and `apply_dedup_plan`) need identical behavior, and because the rewrite must happen **before** `net.retract(old_id)` — otherwise the cascade would already have knocked downstream beliefs OUT.

---

### Contract

**Preconditions** (none enforced by types — all are "assumed true"):

1. `old_id` and `new_id` both resolve in `net.nodes` (otherwise: `KeyError`).
2. `old_id != new_id`. This is *not* checked. If they are equal, `old_node is new_node`; the function would add `dep_id` to the `dependents` set in the rewrite branch and then discard it at the end, leaving the reverse index corrupted (dependent removed while its justification still cites the node).
3. The reverse index (`old_node.dependents`) must be up to date — i.e., it already lists every node whose justifications reference `old_id`. Any stray reference missed by the index will *not* be rewritten.
4. The caller must intend to subsequently retract `old_id` and trigger propagation (e.g., via `net.retract(old_id)`), or the dependent's truth values will not be recomputed.

**Postconditions**:

- For every `dep_id` in `old_node.dependents` **at call entry**:
  - `old_id` no longer appears in any of `dep.justifications[*].antecedents`.
  - `old_id` no longer appears in any of `dep.justifications[*].outlist`.
  - `new_id` appears where `old_id` previously appeared.
  - `dep_id ∈ new_node.dependents`.
  - `dep_id ∉ old_node.dependents`.
- No nodes are added, removed, or re-typed.
- Truth values are **not** recomputed.

**Invariants it maintains**: the reverse index (`node.dependents`) is kept consistent with the forward references (`justification.antecedents` / `justification.outlist`) for the two nodes involved.

---

### Parameters

| Name | Type | Meaning | Edge cases |
|------|------|---------|-----------|
| `net` | `Network` (from `reasons.network`) | The in-memory graph. Must expose `.nodes: dict[str, Node]`; `Node` must expose `.justifications: list[Justification]` and `.dependents: set[str]`. | Assumed live/mutable; no persistence here (the `_with_network(..., write=True)` context around callers does the saving). |
| `old_id` | `str` | The duplicate node about to be retracted. | Must exist; will `KeyError` otherwise. Should not equal `new_id`. |
| `new_id` | `str` | The canonical node being kept. Its `dependents` set grows. | Must exist; `KeyError` otherwise. Assumed IN by the caller (a duplicate is never kept over an OUT one — `deduplicate` only considers IN nodes). |

---

### Return Value

`None`. This is a pure side-effecting mutator. There is no success/failure signal — either it completed and rewired the graph, or it raised.

---

### Algorithm (step-by-step)

```
1. old_node = net.nodes[old_id]          # KeyError if unknown
2. new_node = net.nodes[new_id]          # KeyError if unknown

3. for dep_id in list(old_node.dependents):     # snapshot because we mutate the set
       dep = net.nodes[dep_id]

       for j in dep.justifications:
           if old_id in j.antecedents:
               j.antecedents = [new_id if a == old_id else a   for a in j.antecedents]
               new_node.dependents.add(dep_id)

           if old_id in j.outlist:
               j.outlist = [new_id if o == old_id else o       for o in j.outlist]
               new_node.dependents.add(dep_id)

       old_node.dependents.discard(dep_id)         # unconditional
```

Key details worth flagging:

- **Snapshot via `list(...)`** is essential. Inside the loop we call `old_node.dependents.discard(...)` — iterating a live set would raise `RuntimeError: Set changed size during iteration`.
- **Only iterates direct dependents of `old_node`**. This is sufficient because transitive dependents *stay* dependent on the (now-rewired) direct dependent. The graph topology is preserved; only the leaf references are renamed.
- **Two branches are independent**. A dependent could cite `old_id` in the antecedents of one justification and the outlist of another; both are rewritten in the same pass. `new_node.dependents.add(dep_id)` is called twice but is idempotent (a `set`).
- **Both inlist and outlist are reverse-indexed**. In the `Network` model, outlist references are stored in the referenced node's `dependents` set too (visible in the `export_network` reconstruction, which does `for out_id in j.outlist: ... dependents.add(nid)`), so iterating `old_node.dependents` correctly enumerates outlist consumers as well.
- **List reassignment, not in-place mutation**. `j.antecedents = [ ... ]` produces a *new* list object. Any external holder of the previous list reference will not see the rewrite. In practice, the `Justification` object is the sole owner, so this is fine — but it's worth knowing.
- **No de-duplication**. If a dependent's antecedents were `[old_id, new_id]` (it already cited both), the rewrite yields `[new_id, new_id]`. Semantically harmless for "all antecedents IN" logic but leaves a redundant entry.
- **`discard` is unconditional** for every previously-known dependent — even ones whose justifications didn't actually contain `old_id` (which would indicate stale reverse-index state, cleaned up as a side effect).

---

### Side Effects

- Mutates `Justification.antecedents` (attribute re-bound to a new list) on every dependent's justification that cited `old_id`.
- Mutates `Justification.outlist` similarly.
- Mutates `Node.dependents` sets on **both** `old_node` (removes entries) and `new_node` (adds entries).
- **No** persistence, **no** logging, **no** pub-sub events. The caller's `_with_network(..., write=True)` context handles DB save and change notifications on exit.
- **No** truth-value recomputation — that happens later when the caller invokes `net.retract(old_id)`.

---

### Error Handling

| Condition | Behavior |
|-----------|----------|
| `old_id` not in `net.nodes` | `KeyError` from `net.nodes[old_id]` |
| `new_id` not in `net.nodes` | `KeyError` from `net.nodes[new_id]` |
| A listed dependent is missing from `net.nodes` | `KeyError` from `net.nodes[dep_id]` (indicates a stale reverse index — an inconsistency the function doesn't guard against) |
| `old_id == new_id` | Silent corruption of `dependents` (add-then-discard on the same set), no exception |
| Nothing else is caught, swallowed, or logged. | |

There are no explicit `raise` statements and no defensive checks.

---

### Usage Patterns

Both callers are inside `_with_network(db_path, write=True)` and follow the same sequence:

```python
# in deduplicate(auto=True)
for nid in members:
    if nid != keep:
        _rewrite_dependents(net, old_id=nid, new_id=keep)   # rewires refs to keep
        net.retract(nid)                                     # now the cascade is a no-op on rewired deps
        retracted.append(nid)

# in apply_dedup_plan
for old_id in cluster["retract"]:
    if net.nodes[old_id].truth_value == "OUT":
        continue                                             # already gone, nothing to rewrite
    _rewrite_dependents(net, old_id=old_id, new_id=keep)
    net.retract(old_id)
    retracted.append(old_id)
```

**Caller obligations**:

1. Both nodes present, distinct, and IN (at least `new_id` should be IN).
2. Call `net.retract(old_id)` (or `recompute_all`) immediately after so truth values settle.
3. Run inside a write context so mutations reach storage.
4. Do not rely on `old_node.dependents` afterwards — it was emptied for all previously-known dependents.

---

### Dependencies

- Only the `Network` / `Node` / `Justification` object graph from `reasons.network` / `reasons` — no imports, no SQLite access, no I/O.
- Implicitly depends on `Network.add_node` / `Network.add_justification` maintaining the `dependents` reverse index correctly (otherwise the contract is silently violated).

---

## Topics to Explore

- [file] `tests/test_dependents_integrity.py` — Verifies the reverse-index invariants that `_rewrite_dependents` depends on and repairs.
- [function] `reasons/network.py:Network.retract` — The cascade that runs *after* rewiring; confirms dependents stay IN because references now point at the kept node.
- [function] `reasons/api.py:deduplicate` — Main `auto=True` caller that picks the "most connected" node as `keep` and calls this per duplicate.
- [function] `reasons/api.py:apply_dedup_plan` — Reviewed-plan caller; also uses this helper.
- [general] `reverse-index-consistency` — How `dependents` sets are populated incrementally by `add_node` / `add_justification` and re-hydrated in `export_network` / `import_json`.

---

## Beliefs

- `_rewrite-dependents-requires-consistent-reverse-index` — Only nodes already listed in `old_node.dependents` get their justifications rewritten; any reference missed by the reverse index keeps pointing at `old_id`.
- `_rewrite-dependents-reassigns-justification-lists` — Uses list-comprehension reassignment, so `j.antecedents` and `j.outlist` become new list objects rather than being mutated in place.
- `_rewrite-dependents-processes-antecedents-and-outlist` — Rewrites both `antecedents` and `outlist` entries equal to `old_id`, and mirrors the resulting references into `new_node.dependents`.
- `_rewrite-dependents-does-not-recompute-truth-values` — Truth-value propagation is deferred; callers must invoke `net.retract(old_id)` afterwards for the rewiring to take effect in the belief state.
- `_rewrite-dependents-assumes-old-and-new-are-distinct` — Passing `old_id == new_id` silently corrupts the `dependents` set (adds then discards on the same set); the check is not enforced.
