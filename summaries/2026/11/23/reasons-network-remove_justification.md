---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: remove_justification in reasons/network.py

# `remove_justification` — `reasons/network.py`

## Purpose

`remove_justification` removes a **single** justification (by position) from a node's list of justifications, then recomputes that node's truth value and propagates any downstream change through the dependency graph.

It exists to support *justification editing*: after import or rule construction, a node may end up with more justifications than intended (e.g., redundant rationales, stale dependencies, or a "Depends on:" edge that turned out to be contextual rather than logical). This method lets a caller prune one support without destroying the node, unlike `convert_to_premise` (which nukes all justifications and forces the node to be a premise) or `retract` (which leaves justifications intact but forces OUT).

A key design decision is that it *refuses* to remove the last justification. Doing so would either silently promote the node to a premise (`convert_to_premise`'s job) or leave a derived node with no grounds for IN/OUT (`_compute_truth` on a node with no justifications returns the current truth value, which would be nonsensical for a derived node). Refusing keeps the invariant "if a node has zero justifications, it is a premise."

---

## Contract

**Preconditions**
- `node_id` is a key in `self.nodes`.
- The node has at least one justification (i.e., it is not a premise).
- `0 <= index < len(node.justifications)`.
- The node has **more than one** justification (removing the last one is disallowed).

**Postconditions**
- `node.justifications` has `len(before) - 1` entries.
- If the removed justification was the *only* reference to a given antecedent or outlist node across the remaining justifications, the reverse-index edge `self.nodes[ref].dependents` no longer contains `node_id`.
- `node.supporting_justification` may have been reset by `_compute_truth` (this is *not* explicitly updated in the removed branch — see caveats below).
- If the node's truth value changed, `node.truth_value` is updated and `_propagate(node_id)` has been called to recompute dependents.
- An audit entry has been appended to `self.log` with action `"remove-justification"`.
- A summary dict describing the removal is returned.

**Invariants maintained**
- Reverse index `dependents` mirrors the union of `antecedents` ∪ `outlist` across *live* justifications (as verified by `verify_dependents`).
- Premises (`justifications == []`) are not modified by this method (rejected by the count check).
- Node objects persist in `self.nodes` (no removal from network); only the justification entry is pruned.

---

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `node_id` | `str` | ID of the node whose justification should be dropped. | Raises `KeyError` if not in `self.nodes`. |
| `index` | `int` | Zero-based position into `node.justifications`. | Negative indices are **rejected** (`index < 0`); Python's native "negative means from end" behavior is disabled on purpose. Out-of-range raises `IndexError`. |

There is no support for removing by identity, by label, or by a predicate — only positional removal. Callers who want to remove "the justification that depends on node X" must find its index themselves.

---

## Return Value

Returns a plain `dict` with these keys:

| Key | Type | Meaning |
|---|---|---|
| `node_id` | `str` | Echo of the input target. |
| `old_truth_value` | `"IN"` or `"OUT"` | Truth value *before* removal. |
| `new_truth_value` | `"IN"` or `"OUT"` | Truth value after recomputation (may equal `old`). |
| `removed` | `dict` | Snapshot of the removed justification: `type`, `antecedents`, `outlist`, `label`. Note `antecedents`/`outlist` are **the live lists from the Justification object**, not copies. |
| `remaining` | `int` | Count of justifications left on the node. |
| `changed` | `list[str]` | IDs whose truth value changed as a result. Includes `node_id` first if it changed, then any downstream dependents from `_propagate`. Empty if nothing changed. |

### Caveat on `removed`
`removed_dict["antecedents"]` and `removed_dict["outlist"]` are aliases to the popped justification's lists. Since the justification object has been popped from `node.justifications`, no mutation path through the network will touch it, but a caller holding the returned dict shares mutable state with the (now-detached) `Justification` object.

---

## Algorithm

1. **Validate target existence.** `KeyError` if `node_id` is missing.
2. **Validate premise check.** If `node.justifications` is falsy, raise `ValueError` — premises have nothing to remove.
3. **Validate index.** Explicit `index < 0 or index >= len(...)` guard raises `IndexError`. Note this rejects `-1` even though Python list indexing would accept it; the author deliberately chose positional-only semantics.
4. **Validate "not the last one."** If there is exactly one justification, raise `ValueError` pushing caller toward `convert-to-premise` or `retract`. This prevents accidentally turning a derived node into an inconsistent state where `_compute_truth` has nothing to evaluate against.
5. **Capture `old_value`**.
6. **Pop the justification** at `index`. `removed` is the popped object.
7. **Recompute `remaining_refs`**: union of `antecedents` and `outlist` across *all remaining* justifications. This is the set of nodes that still need a `dependents` edge from `node_id`.
8. **Prune stale reverse-index edges**: for each `ant_id` in `removed.antecedents`, if it is not in `remaining_refs` and exists in `self.nodes`, drop `node_id` from `self.nodes[ant_id].dependents`. Then do the same for each `out_id` in `removed.outlist`. Note: `outlist` nodes are also registered in `dependents` because they can flip the node's truth value (SL/CP semantics check `outlist` for OUT-ness).
9. **Recompute truth** via `_compute_truth(node)`. That helper scans the remaining justifications in **reverse insertion order** and picks the last valid one (setting `node.supporting_justification` to its index), or returns OUT if none are valid.
10. **Propagate if changed**: if `old_value != new_value`, write `node.truth_value`, append `node_id` to `changed`, and extend with `_propagate(node_id)`.
11. **Log** an entry via `_log("remove-justification", node_id, new_value)` — always logged, even when nothing changed.
12. **Return** the summary dict.

### Subtle points to notice in step 8
- The pruning uses only the **removed** justification's refs to decide candidates; a ref that appears in *another* justification *other than the removed one* is kept — but a ref that appears **only inside the removed justification** is dropped.
- The check `ant_id in self.nodes` guards against dangling references. If a justification referenced a node that was since deleted from `self.nodes`, this silently skips it (no error, no warning).
- The same node may appear in both `antecedents` and `outlist` of the *removed* justification — it will be processed twice; the second `discard` is a no-op.

### Subtle points in step 9 (about `supporting_justification`)
`_compute_truth` sets `node.supporting_justification = i` if some remaining justification is valid. Since removal can shift indices, any caller who previously stored the index of the "currently supporting" justification will see a different index after this operation. There's no attempt here to remap the stored index; it is simply recomputed by `_compute_truth`.

---

## Side Effects

All effects are **in-memory mutations** on the `Network` instance:

1. **`node.justifications`**: one element popped.
2. **Reverse index** (`self.nodes[ref].dependents`): stale edges `ref -> node_id` discarded for refs no longer appearing in any remaining justification.
3. **`node.truth_value`**: possibly updated.
4. **`node.supporting_justification`**: possibly updated/reset by `_compute_truth`.
5. **Downstream nodes**: any dependents' `truth_value`/`supporting_justification`, via `_propagate`'s BFS (`_compute_truth` on each).
6. **`self.log`**: one `"remove-justification"` entry appended, plus one `"propagate"` entry per dependents that changed.
7. **Callback chain**: `_propagate` may append further `"propagate"` entries via `_log`.

No I/O, no file writes, no time capture (other than `_log` timestamps from the `datetime` module).

Note: unlike `add_justification`, this method does **not** recompute `content_hash` on anything. Hashes of the removed justification are discarded with it; remaining justifications keep their hashes (they are unchanged).

---

## Error Handling

| Situation | Exception | Message |
|---|---|---|
| Node unknown | `KeyError` | `Node '<id>' not found` |
| Node is a premise | `ValueError` | `Node '<id>' is a premise (no justifications)` |
| Index out of range (including negatives) | `IndexError` | `Justification index N out of range (node has M)` |
| Only one justification remains | `ValueError` | Suggests `convert-to-premise` or `retract` |

Swallowed / silent:
- Missing `ant_id`/`out_id` in `self.nodes` during reverse-index cleanup — no error, no warning, just skipped.
- If `_compute_truth` returns the same value as `old_value`, no propagation is performed. This is intentional (avoids spurious graph walks), but means a caller who wants "recompute everything because of potential hash/metadata drift" will need `recompute_all()` instead.

No exception is *converted* into a returned error dict; all failures raise.

---

## Usage Patterns

Typical call sequence:

```python
result = net.remove_justification("hyp-42", index=1)
if result["changed"]:
    # propagation touched downstream nodes
    ...
```

Callers should:
- **Be aware of index shifting.** After removal, any stored index into `justifications` (including from `supporting_justification`) changes meaning for indices ≥ the removed one. Prefer re-querying over caching.
- **Not treat `changed == []` as "no-op."** The justification *was* removed even if the truth value and downstream values did not change (e.g., the removed justification was redundant).
- **Use `convert_to_premise`** if they want to strip all justifications and make the node a premise; **`retract`** if they want to force OUT without touching justifications.
- **Check `remaining`** if it matters whether the node is at risk of the "only one justification" guard.
- **Handle exceptions** with a `try/except` per case above if inputs are user-supplied (e.g., from a CLI).

### Assumptions not enforced by types
- `index` is presumed to be a plain `int`; a `bool` would be accepted (since `bool` subclasses `int` in Python) and would act as 0/1.
- The method assumes `node.justifications` is a `list[Justification]` and that elements expose `.type`, `.antecedents`, `.outlist`, `.label`. This is common across the file but not enforced by type hints on `Node` (see the `Node` dataclass definition in `reasons/__init__.py`).
- It assumes antecedents and outlist are iterable collections of `str` node IDs that may reference nodes already removed from the network — hence the `in self.nodes` guards.
- No locking / thread-safety; concurrent callers can race on `dependents` sets and `self.log`.

---

## Dependencies

- **`Node`** (from `.`): provides `.justifications`, `.truth_value`, `.supporting_justification`, `.dependents`, `.metadata`.
- **`Justification`** (from `.`): the popped object's shape (`type`, `antecedents`, `outlist`, `label`, `content_hash`).
- **`self._compute_truth`**: internal helper that evaluates validity of remaining justifications.
- **`self._propagate`**: internal BFS over `dependents` reaching a fixpoint on downstream truth values.
- **`self._log`**: internal audit-trail writer, uses `datetime.now()` from stdlib.
- **No external packages** beyond stdlib (`datetime` via `_log`).

---

## Topics to Explore

- [function] `reasons/network.py:_compute_truth` — Explains why removing a justification can flip a node from IN to OUT, and how `supporting_justification` is chosen (reverse insertion order, last valid wins).
- [function] `reasons/network.py:_propagate` — BFS over `dependents`; the correctness of `changed` in the return value depends entirely on this crawl.
- [function] `reasons/network.py:convert_to_premise` — The intended alternative when the last justification must go; contrasts how the invariant "no justifications ⇒ premise" is preserved.
- [file] `tests/test_remove_justification.py` — Ground truth for edge cases (index ranges, single-justification guard, dependents cleanup).
- [general] `reverse-dependents-invariant` — The `dependents` set is reconstructed by `_rebuild_dependents`; `verify_dependents` asserts it matches the justification graph. This function's step-8 cleanup is one of only a few places that perform the discard manually.

---

## Beliefs

- `remove-justification-guard-prevents-last-removal` — The method raises `ValueError` when a node has exactly one justification; callers must use `convert_to_premise` or `retract` instead.
- `remove-justification-is-positional-only` — Negative `index` values are rejected; therefore callers cannot use Python slice-style `-1` to remove the last justification.
- `remove-justification-propagates-only-on-truth-change` — If `_compute_truth` returns the same value as `old_value`, no BFS propagation occurs and `changed` excludes downstream nodes even if their metadata or support indices shifted.
- `remove-justification-cleans-stale-dependents` — For each antecedent/outlist in the removed justification that no longer appears in any remaining justification, `node_id` is discarded from that node's `dependents` set; if the ref is missing from `self.nodes` the discard is silently skipped.
- `remove-justification-leaves-justification-hashes-untouched` — Unlike `add_justification`, this method does not recompute `content_hash` or `text_hash`; the returned `removed` dict aliases the popped object's original lists.
