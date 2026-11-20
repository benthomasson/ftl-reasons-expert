---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: defeat_with_scope in reasons/api.py

# `defeat_with_scope` — Explanation

## Purpose

`defeat_with_scope` is a **reversible, proof-backed** variant of `defeat_justification`. Where the sibling function creates a **bare premise defeater** (a node with no justifications that is always IN), `defeat_with_scope` instead:

1. Creates a set of **scope belief** premises describing what each antecedent of the target justification actually establishes / does not establish.
2. Creates a **derived defeater node** whose SL justification cites all of those scope beliefs.
3. Puts that defeater node on the target justification's `outlist`, causing the target to go OUT.

Because the defeater is derived (not a premise), it is only IN while **every** scope belief is IN. Retracting or challenging any one scope belief flips the defeater OUT, which removes the outlist block and lets the target belief be restored by normal TMS propagation. This gives the system a **surgical, reversible** retraction knob instead of an unconditional premise defeater.

## Contract

**Preconditions (enforced):**
- `node_id` exists in the network.
- The node has at least one justification (it is not a premise).
- `justification_index ∈ [0, len(node.justifications))`.
- `scope_findings` is a non-empty list of dicts, each with at minimum an `"antecedent"` key.

**Preconditions (assumed, not enforced by the type system):**
- Each `scope_findings[i]["antecedent"]` should correspond to a real antecedent of the target justification. This is **not validated** — the antecedent ID only appears in the scope node's ID and text; it is not wired into the scope belief as a justification dependency.
- `defeater_id` (if provided) does not already exist in the network. The function does not pre-check for a collision — it relies on `Network.add_node`'s behavior.
- Elements of `scope_findings` have a stable, consistent shape (e.g., not repeating the same `antecedent` with conflicting `establishes` values).

**Postconditions (on success):**
- One new node per `scope_findings` entry is created (premises), returned in `scope_belief_ids` in input order.
- One new derived defeater node is created with an SL justification over those scope beliefs.
- The defeater ID is appended to `node.justifications[justification_index].outlist` (idempotent — guarded by a membership check).
- The defeater's `dependents` reverse index includes `node_id`.
- `net.recompute_all()` has been run and the target's truth value reflects the new outlist.
- The network is saved to disk and pubsub events are emitted (via `_with_network(write=True)`).

**Invariant:** if any scope belief is later retracted/challenged, the defeater's SL justification becomes unsatisfied, defeater goes OUT, and the target's outlist block disappears — restoring the original belief (assuming another justification doesn't hold).

## Parameters

| Name | Type | Meaning | Notes |
|---|---|---|---|
| `node_id` | `str` | The belief whose justification is being defeated. | Must exist; validated. |
| `justification_index` | `int` | 0-based index into `node.justifications`. | Bounds-checked; negative indices excluded. |
| `scope_findings` | `list[dict]` | Per-antecedent scope claims. Each needs `antecedent` (required), `establishes` (optional), `does_not_establish` (optional). | Empty list raises `ValueError`. |
| `missing_property` | `str` | What the derived belief claims but no antecedent establishes. Baked into the defeater text. | Used only in the defeater's `text`, no semantic role in the graph. |
| `defeater_type` | `str` = `"invalid-inference"` | Category tag on the defeater node's metadata. | Also forms part of the default defeater ID. |
| `defeater_id` | `str \| None` | Custom defeater node ID. | Default: `f"{defeater_type}-{node_id}-j{justification_index}"`. Collisions not pre-checked. |
| `defeat_reason_type` | `str` = `""` | Logical failure mode classification, stored on metadata only if non-empty. | Echoed back in the result. |
| `db_path` | `str` = `DEFAULT_DB` | SQLite database path. | Passed to `_with_network`. |

**Edge cases:**
- Empty `scope_findings` → `ValueError`.
- `justification_index` out of range (`<0` or `>=len`) → `IndexError`.
- Missing `"antecedent"` key inside a scope finding → `KeyError` mid-loop; the write context's `__exit__` sees `exc_type is not None`, so **nothing is saved** and the cache is not updated. Atomic at the process level.
- Two scope findings sharing an `antecedent` produce two scope belief nodes; the second ID generation gets a `-1`, `-2`, … suffix.
- If `defeater_id` is already in the target's outlist (previously defeated the same way), the membership guard avoids duplicate outlist entries — but `net.add_node` for the defeater itself may still error on duplicate ID. Behavior depends on `Network.add_node`.

## Return Value

Always a `dict` on success:

```python
{
    "node_id": str,                 # target belief
    "justification_index": int,     # echo of input
    "defeater_id": str,             # actual defeater ID used/created
    "defeater_type": str,           # echo
    "defeat_reason_type": str,      # echo ("" if not supplied)
    "scope_belief_ids": list[str],  # IDs of created scope premise nodes, in input order
    "changed": list[str],           # nodes whose truth_value actually flipped
}
```

**Caller obligations:**
- `changed` can be empty when the target stays IN despite the defeat — e.g., another justification still satisfies it. Never assume the defeat worked just because no exception was raised; check `changed`.
- The returned `scope_belief_ids` are the handles for reversal: retracting or challenging one is how to undo the defeat.
- Exceptions (`KeyError`, `ValueError`, `IndexError`) are the failure signal; no partial writes occur on error.

## Algorithm

1. **Enter write context.** `with _with_network(db_path, write=True)` — loads fresh (or uses cached on write-fallback), and snapshots truth values only if pubsub subscribers exist.
2. **Validate target.** `node_id ∈ net.nodes`, else `KeyError`. Node must have `justifications`, else `ValueError`.
3. **Bounds-check index.** Reject negatives and out-of-range.
4. **Resolve defeater ID.** If `defeater_id` is `None`, derive one from `defeater_type`, `node_id`, and `justification_index`.
5. **Guard scope findings.** `if not scope_findings: raise ValueError`.
6. **Snapshot before-state.** `before = {nid: truth_value}` for every node — used later to compute `changed`.
7. **Materialize scope beliefs.** For each `sf`:
   - Read `sf["antecedent"]` (KeyError if present).
   - Generate `scope_id = f"scope-{ant_id}-for-{node_id}-j{idx}"`, suffixing `-1`, `-2`, … on collision with existing nodes.
   - Compose `scope_text` from `establishes` and optionally `does_not_establish`.
   - `net.add_node(id=scope_id, text=scope_text, metadata={"scope_of": ant_id, "for_defeater": defeater_id})`. Note: **no justifications** → premise.
   - Append `scope_id` to `scope_belief_ids`.
8. **Construct defeater.** Build `defeater_text` referencing `node_id`, `missing_property`, `justification_index`. `net.add_node(id=defeater_id, text=defeater_text, justifications=[Justification(type="SL", antecedents=scope_belief_ids, label=...)], metadata={defeater_type, defeats_node, defeats_justification, [defeat_reason_type]})`.
9. **Attach outlist.** If `defeater_id ∉ justification.outlist`, append. Register the reverse dependency: `defeater.dependents.add(node_id)`.
10. **Touch timestamp.** `node.updated_at = now`.
11. **Recompute.** `changed = net.recompute_all()`. Because the defeater is IN (all scope premises IN), the target's justification is now outblocked.
12. **Diff.** Keep only IDs where `before[nid] != net.nodes[nid].truth_value` — this is `actually_changed` (top-level propagation may also touch dependents of the target).
13. **Exit write context.** Saves `net` via `Storage.save`, then if subscribers exist, diffs `_before` vs current and calls `pubsub.publish`.
14. **Return dict.**

## Side Effects

- **Node creation:** N scope belief premise nodes + 1 derived defeater node.
- **Justification mutation:** appends `defeater_id` to the target justification's `outlist` in place.
- **Reverse index mutation:** `defeater.dependents` gains `node_id`.
- **Timestamp mutation:** target node's `updated_at` set to current UTC ISO timestamp.
- **Truth-value cascade:** `recompute_all()` may flip the target and its dependents between IN and OUT.
- **Persistence:** SQLite write via `Storage.save` (only committed when no exception occurred inside the context).
- **Pub/Sub:** if `pubsub.has_subscribers(db_path)`, publishes node truth-value change events computed from the before/after snapshot taken in `__enter__`.
- **Cache:** `_network_cache[db_path]` is refreshed on clean exit.

## Error Handling

| Situation | Raised |
|---|---|
| `node_id` not found | `KeyError` |
| Node has no justifications | `ValueError` |
| `justification_index` out of range | `IndexError` |
| `scope_findings` empty | `ValueError` |
| Scope finding missing `"antecedent"` | `KeyError` |
| `defeater_id` collides (via `net.add_node`) | Depends on `Network.add_node` — likely `ValueError`/`KeyError` |
| Scope ID collisions | **Swallowed** — a numeric suffix is appended; new node is created under the suffixed ID |

No return-value-level error channel — the only "soft" signal is an empty `changed` list. On any raised exception inside the context, `_with_network.__exit__` skips persistence and skips cache update (the write context only saves when `exc_type is None`).

## Usage Patterns

Typical callers are the CLI / LangGraph tool layer and review pipelines that have identified a "smuggled premise" — a derived belief whose conclusion asserts a property not actually established by its antecedents. The caller must:

1. Supply `scope_findings` describing, per-antecedent, what the antecedent really does establish and does not establish. The antecedent IDs are used only for the scope node's ID/text; the caller (or human reviewer) must accept this naming convention.
2. Choose `justification_index` explicitly — the target may have multiple justifications; defeat only the offending one.
3. Inspect `changed` to confirm the target actually went OUT.
4. Later, to reverse the defeat: `retract_node(scope_id)` or `challenge(scope_id, ...)` on one of `scope_belief_ids`.

Because this function opens its own `_with_network` context, it **must not** be called from inside an already-open write context (nested `_with_network` would load a second copy of the network and could clobber writes). Notice `migrate_retract_to_defeaters` explicitly inlines defeat logic for exactly this reason.

## Dependencies

- `_with_network(db_path, write=True)` — local context-manager wrapper that handles load/save/cache/pubsub.
- `Justification` — imported from `reasons/__init__.py`; used to build the defeater's SL justification.
- `Network.add_node`, `Network.recompute_all`, `Node.justifications`, `Node.outlist`, `Node.dependents` — the graph primitives this function manipulates directly.
- `pubsub.compute_changes` / `pubsub.publish` — invoked by `_with_network` on write.
- `datetime.now(timezone.utc).isoformat(timespec="seconds")` — timestamp source.
- `Storage` — persistence layer used by `_with_network`.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — how write-mode caching, saving, pubsub, and exception-time skipping of persistence work together
- [function] `reasons/api.py:defeat_justification` — the unconditional premise-defeater sibling; contrast the reversibility tradeoff
- [function] `reasons/network.py:Network.recompute_all` — the propagation engine that turns the outlist change into a cascade
- [file] `reasons/pubsub.py` — how truth-value diffs are computed and delivered to subscribers after a write
- [general] `outlist-semantics` — how TMS outlists interact with multiple justifications and reversibility

## Beliefs

- `defeat-with-scope-creates-derived-defeater` — The defeater node created by `defeat_with_scope` has an SL justification over scope beliefs; it is not a premise, which is what makes the defeat reversible.
- `defeat-with-scope-does-not-link-antecedents` — Scope belief nodes are created as premises with no justification dependency on the `antecedent` node referenced in their ID and text.
- `defeat-with-scope-requires-nonempty-scope-findings` — Passing an empty `scope_findings` list raises `ValueError` before any mutation.
- `defeat-with-scope-is-atomic-on-exception` — Any exception raised inside the write context prevents `Storage.save` and cache updates, because `_with_network.__exit__` only persists when `exc_type is None`.
- `defeat-with-scope-emits-pubsub-events-on-save` — When pubsub subscribers exist, the write path computes truth-value changes from the `_before` snapshot and publishes them after a successful save.
