---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: add_justification in reasons/network.py

# `add_justification` — `reasons/network.py`

## Purpose

`add_justification` adds a new `Justification` object to an *existing* node in the truth-maintenance network and then re-derives that node's truth value, propagating any change through the dependency graph.

It exists because the TMS model is non-monotonic and multi-justification: a node is `IN` iff *at least one* of its justifications is currently valid (see `_compute_truth`). So the ability to bolt on another justification after node creation — for backup arguments, defeaters, supersession evidence, imported context — is a first-class operation distinct from `add_node`. It is the counterpart to `remove_justification` and `convert_to_premise`.

The docstring ("and propagate") understates what happens: there are **two** propagation passes here — one for access tags and one for truth values — and only the second one is reflected in the return value.

---

## Contract

**Preconditions (enforced):**
- `node_id` must exist in `self.nodes`; otherwise `KeyError`.
- `justification` is expected to be a live `Justification` instance (not enforced — duck typing will fail later).

**Preconditions (assumed, *not* enforced):**
- The caller is not passing the *same* `Justification` instance that is already attached to this or another node. There is no deduplication.
- The caller wants the new justification to participate in the node's truth computation regardless of `node.metadata["_retracted"]`. This function does **not** check the retracted flag (unlike `recompute_all`, whose loop skips `_retracted` nodes).
- IDs appearing in `justification.antecedents` / `justification.outlist` may or may not exist in the network; unknown IDs are silently skipped for the reverse-index update.
- Cross-node consistency of `Justification.content_hash` is the caller's concern; the function only fills it in when missing.

**Postconditions:**
- `justification` is in `node.justifications` (appended; order matters because `_compute_truth` prefers the *newest* valid justification).
- For every existing `ant_id` in `justification.antecedents` and every existing `out_id` in `justification.outlist`, `node_id ∈ self.nodes[those_ids].dependents`.
- `justification.content_hash` is populated, and `node.text_hash` is populated if it was missing.
- `node.metadata["access_tags"]` reflects a merge of all antecedent access tags, and dependents have been updated accordingly.
- `node.truth_value` and `node.supporting_justification` are recomputed from the current justification set.
- If the truth value changed, all transitively affected dependents have been updated by `_propagate` (BFS).
- A log entry `("add-justification", node_id, new_value)` is appended to `self.log`.

**Invariants preserved:** the reverse `dependents` index remains a superset-or-exact-match of what the justifications imply (the function only *adds* to it, never removes — removal is `remove_justification`'s job).

---

## Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `node_id` | `str` | ID of an existing node to attach the justification to | Must be present; raises `KeyError` otherwise. No validation of emptiness. |
| `justification` | `Justification` | The justification to attach | Not validated for `.type in {"SL","CP"}`. Unknown antecedent/outlist IDs are tolerated. `content_hash` may be empty and will be filled in. |

---

## Return Value

Always a `dict` with exactly these keys:

```
{
  "node_id":          str,      # same as input
  "old_truth_value":  "IN"|"OUT",  # snapshot before mutation
  "new_truth_value":  "IN"|"OUT",  # after recompute
  "changed":          list[str],   # node_id first (if changed), then BFS-affected dependents
}
```

Notes on the return contract:
- **`changed` is empty if the truth value did not change.** This is common when adding a redundant or invalid justification to a node that is already `IN`.
- **`changed` does *not* include access-tag-only changes.** `_propagate_access_tags` returns nothing; the caller cannot tell from this dict whether downstream nodes saw tag updates.
- **`supporting_justification` may change even when `old_truth_value == new_truth_value`** (`_compute_truth` always reassigns it). This is not surfaced either.
- The caller must inspect `old_truth_value != new_truth_value` to distinguish "no effect" from real propagation.

---

## Algorithm

1. **Validate target node** — `node_id in self.nodes` or raise `KeyError("Node '<id>' not found")`.
2. **Snapshot** `old_value = node.truth_value`.
3. **Update reverse index.** For each `ant_id` in `justification.antecedents` and each `out_id` in `justification.outlist` that exists in the network, add `node_id` to that node's `dependents` set. Missing IDs are silently ignored (they may be added later via `add_node`). *Note: outlist nodes participate in the reverse index because they can drive the node OUT.*
4. **Attach the justification** — `node.justifications.append(justification)`.
5. **Populate Merkle hashes lazily** (avoiding circular imports via function-local imports):
   - If `justification.content_hash` is falsy:
     - Ensure `node.text_hash` is present (`compute_text_hash(node.text)` if missing).
     - Set `justification.content_hash = compute_content_hash(node, justification, self)`.
   - No hashing is done if `content_hash` already set — useful for content-addressed reuse, but it means a hashed `Justification` shared between nodes carries a hash tied to whichever node computed it first.
6. **Access-tag propagation (two phases):**
   - `_inherit_access_tags(node)` merges `access_tags` from every antecedent the node currently has a justification for into `node.metadata["access_tags"]`.
   - `_propagate_access_tags(node_id)` walks the `dependents` graph forward (BFS, `deque`, `visited` set) and calls `_inherit_access_tags` on each dependent, re-enqueueing only when the dependent's tag set actually changed.
7. **Recompute truth value** via `_compute_truth(node)`. That helper:
   - Returns the current value if `node.justifications` is empty (irrelevant here — we just added one).
   - Iterates the justification list from newest to oldest, picking the first one where `_justification_valid` returns `True`; sets `node.supporting_justification` to that index and returns `"IN"`. If none are valid, sets `supporting_justification = None` and returns `"OUT"`.
8. **Propagate truth-value changes only if the value changed.** `changed = [node_id]`, then `changed.extend(self._propagate(node_id))`. `_propagate` is BFS through `dependents`, skipping already-visited nodes and any dependent whose `metadata["_retracted"]` is truthy, calling `_compute_truth` on each; re-enqueues only on truth change.
9. **Log** — `self._log("add-justification", node_id, new_value)`.
10. **Return** the dict described above.

---

## Side Effects

In mutation order:

- `self.nodes[ant_id].dependents ∪= {node_id}` for each existing antecedent.
- `self.nodes[out_id].dependents ∪= {node_id}` for each existing outlist node.
- `node.justifications += [justification]`.
- `node.text_hash = ...` (conditional).
- `justification.content_hash = ...` (mutates the caller-supplied object).
- `node.metadata["access_tags"] = sorted(merged_set)` (conditional — inside `_inherit_access_tags`, only if the inherited set is non-empty).
- Recursive updates to dependent nodes' `metadata["access_tags"]` via `_propagate_access_tags`.
- `node.truth_value` and `node.supporting_justification`.
- Cascade through all coherent dependents' `truth_value`.
- Append one entry to `self.log`.

**No I/O**, no external calls beyond `.merkle`'s hashing helpers.

---

## Error Handling

| Condition | Behavior |
|---|---|
| Unknown `node_id` | `raise KeyError(...)` |
| Unknown antecedent/outlist ID | Silently skipped in the reverse-index loop. Justification itself still references the missing ID, and `_justification_valid` treats unknown antecedents as `OUT` (invalidating the justification) and unknown outlist nodes as `OUT` (satisfying the outlist requirement). |
| `compute_content_hash` / `compute_text_hash` failures | Not caught — propagate to caller. |
| `add_justification` on a node flagged `_retracted` | **No error, no warning.** `_compute_truth` will still run and may flip the node back to `"IN"` if the new justification is valid. This bypasses the retraction flag that `recompute_all` respects; whether intentional or not, callers should be aware. |
| Same justification instance added twice | No error; the `dependents` sets are idempotent, but `node.justifications` grows by one, and `_compute_truth` may select the duplicate as `supporting_justification`. |
| Same justification added to two nodes | `content_hash` is computed only the first time; the second node receives the first node's hash. |

---

## Usage Patterns

- **Typical call**: `network.add_justification("claim-42", Justification(type="SL", antecedents=["evidence-1"], outlist=["counter-3"]))`.
- **Suggested caller flow**:
  1. Ensure all antecedents/outlist nodes exist *first* if you want the reverse index and truth computation to be complete — otherwise the justification is stored but treated as invalid.
  2. Inspect `result["changed"]` to know whether cascades occurred.
  3. Inspect `result["old_truth_value"]` vs `result["new_truth_value"]` to know whether the node itself flipped.
  4. If access-tag propagation matters, inspect downstream nodes' `metadata["access_tags"]` separately — the return dict does not report it.
- **When *not* to use it**: for initial justification at node creation use `add_node`; for reversing a previous add use `remove_justification`; for defeating/superseding a node use `challenge` / `defend` / `supersede` (they manage the outlist and `dependents` index with their own logic). Do **not** use this for adding a justification that references nodes which will *never* exist — `_justification_valid` will silently treat unknown antecedents as `OUT` and the node may go `OUT` unexpectedly.
- **Failure atomicity**: if a later step (e.g., `_propagate`) raises, earlier mutations are **not** rolled back. Callers needing transactional semantics must snapshot and restore themselves.

---

## Dependencies

- **Module-level**: `Node`, `Justification`, `Nogood` from `reasons` package; `collections.deque`, `datetime`, `timezone`.
- **Function-local (lazy to avoid circular imports with `merkle.py`)**:
  - `.merkle.compute_content_hash(node, justification, network)`
  - `.merkle.compute_text_hash(node.text)`
- **Internal helpers consumed**:
  - `_inherit_access_tags(node)` — pulls tags from antecedents
  - `_propagate_access_tags(node_id)` — BFS for tag changes
  - `_compute_truth(node)` — returns `"IN"|"OUT"`, sets `supporting_justification`
  - `_propagate(node_id)` — BFS truth-value propagation
  - `_justification_valid(j)` — used indirectly via `_compute_truth`
  - `_log(action, target, value)`

---

## Assumptions Not Enforced by the Type System

1. `justification` is genuinely a `Justification`; a duck-typed object missing `.antecedents` hints will fail inside `_compute_truth`.
2. `node.justifications` does not already contain this exact instance.
3. `justification.content_hash`, if pre-set, is correct for *this* node (it is trusted as-is and not recomputed).
4. The caller understands that the `_retracted` metadata flag is bypassed.
5. The caller knows `supporting_justification` may change silently.
6. The reverse index and the justification graph are kept consistent by *routing additions through this method* — direct `node.justifications.append(...)` would corrupt `dependents`.

---

## Topics to Explore

- [function] `reasons/network.py:_compute_truth` — Understand "newest valid justification wins" and how `supporting_justification` is chosen; this affects `explain`.
- [function] `reasons/network.py:_justification_valid` — Contract for SL/CP semantics; unknown IDs behave asymmetrically between inlist and outlist.
- [file] `reasons/merkle.py` — How `compute_content_hash` and `compute_text_hash` derive values and what the `network` argument is used for.
- [file] `tests/test_add_justification.py` — Concrete expectations, including dedup, hashing, and access-tag propagation scenarios.
- [function] `reasons/network.py:remove_justification` — The paired reverse operation and how `remaining_refs` cleanup avoids stale `dependents` entries; contrast with `_rebuild_dependents`.
- [general] `retracted-flag-consistency` — Where `metadata["_retracted"]` is honored vs bypassed (`recompute_all` vs `add_justification`), and how that affects restoration semantics.

---

## Beliefs

- `add-justification-does-not-deduplicate` — The function never checks whether an equivalent justification is already attached; duplicates are appended and can become `supporting_justification`.
- `add-justification-silently-skips-unknown-node-ids` — Antecedent/outlist IDs absent from `self.nodes` are ignored for the `dependents` index but remain in the justification, where `_justification_valid` treats them as `OUT`.
- `add-justification-ignores-retracted-flag` — Unlike `recompute_all`, this function does not check `node.metadata["_retracted"]` before recomputing truth, so adding a valid justification can silently flip a retracted node back to `IN`.
- `add-justification-return-omits-tag-changes` — The returned `changed` list contains only truth-value changes; `_propagate_access_tags` effects are not surfaced to the caller.
- `add-justification-may-change-supporting-justification-without-new-truth-value` — `_compute_truth` reassigns `node.supporting_justification` on every call, even when `old_truth_value == new_truth_value`, and the return dict does not expose this.
- `add-justification-lazily-imports-merkle` — `.merkle` imports are performed inside the function to avoid a circular import at module load time.
