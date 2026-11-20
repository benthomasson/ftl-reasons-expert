---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: mark_superseded in reasons/api.py

# `mark_superseded` — `reasons/api.py`

## Purpose

Retracts an obsolete node (`old_id`) and records a permanent metadata link to its replacement (`new_id`) via `metadata["superseded_by"]`. It's a **"hard" supersession**: the old node goes OUT because a string-based `retract_reason` flag is set, *not* because an outlist justification is defeated.

This is a **legacy-style** function. Compare it to the more modern `supersede()` (which uses `net.supersede()` and the graph-native outlist mechanism, so the retraction is reversible — if `new_id` ever goes OUT, `old_id` comes back IN) and to `supersede_with_text()` (which also creates the replacement node). `mark_superseded()` is the simplest and most brittle of the three: it requires both nodes to already exist, and it retracts via `net.retract()` with a string reason.

The same pattern (retract + metadata pointer) exists in `mark_duplicate()` for the dedup case — both are migration targets of `migrate_retract_to_defeaters()`, which exists specifically to convert this style of metadata-based retraction into graph-native defeaters.

## Contract

**Preconditions**
- Both `old_id` and `new_id` must already exist in the network.
- `old_id != new_id`.
- The database file must be writable.
- The caller must accept that the retraction is *irreversible through the TMS* — restoring requires `assert_node` or an explicit metadata edit. There is no outlist-based "un-supersede."

**Postconditions (on success)**
- `net.nodes[old_id].metadata["superseded_by"] == new_id`
- `net.nodes[old_id].metadata["retract_reason"] == "Superseded by {new_id}"`
- `net.retract(old_id, reason=...)` has been called, so `old_id`'s truth value was recomputed (typically OUT) and this cascades to dependents.
- The database was saved via the `_with_network(..., write=True)` context exit.
- No new nodes, justifications, or nogoods were created.
- `new_id` is **not modified** by this function.

**Invariants**
- Node set is unchanged (`len(net.nodes)` is identical before and after).
- `new_id`'s truth value is untouched — the function never asserts `new_id` to IN. If `new_id` is OUT, `old_id` is still tagged as superseded by an OUT node.

## Parameters

| Param | Type | Notes |
|-------|------|-------|
| `old_id` | `str` | Must exist in the network. Full node ID including any namespace prefix (e.g. `"agent:belief-1"`). No namespace resolution happens here. |
| `new_id` | `str` | Must exist in the network. Must differ from `old_id`. No namespace resolution. |
| `db_path` | `str` | Defaults to `DEFAULT_DB` (`"reasons.db"`). Must point to a valid SQLite RMS DB. |

**Edge cases** the signature does **not** enforce:
- Whether `new_id` is actually "newer" or "better" — semantic contract only.
- Whether `new_id` is IN — it can be OUT.
- Whether `new_id` is in the same namespace as `old_id`.
- Whether `new_id` is already `superseded_by` yet another node (chains allowed by design but never validated here).

## Return Value

```python
{
    "old_id": str,        # echoes input
    "new_id": str,        # echoes input
    "changed": list[str], # IDs whose truth value actually changed
}
```

- `changed` is whatever `net.retract()` returns — the cascade of nodes whose truth value moved. This **normally includes `old_id` itself**. Callers should not assume `old_id not in changed`.
- If `old_id` was already OUT before the call (possible if metadata was pre-populated), `changed` may be empty even though metadata was updated.

## Algorithm

```
1. Guard: if old_id == new_id → raise ValueError.
2. Open write context: `with _with_network(db_path, write=True) as net:`
   - Loads a fresh Network from disk (writes bypass the cache).
   - Captures `_before` truth values for pub/sub change detection.
3. Existence check: `old_id` in net.nodes, else KeyError.
4. Existence check: `new_id` in net.nodes, else KeyError.
5. Grab old node. `meta = node.metadata or {}` (defensive default).
6. Mutate metadata in place:
     meta["superseded_by"] = new_id
     meta["retract_reason"] = f"Superseded by {new_id}"
   Then reassign `node.metadata = meta` (belt-and-suspenders; `node.metadata` was
   already `meta` in most cases).
7. Cascade: `changed = net.retract(old_id, reason=reason)`
   - Sets `_retracted`/`retracted_at` flags, recomputes dependents, returns changed IDs.
8. Build and return the result dict. The `with` block's `__exit__` then saves
   to disk and fires pub/sub events if anything changed.
```

## Side Effects

- **Disk write**: The entire network is re-serialized via `Storage.save()` on context exit.
- **Metadata mutations**: Two keys added to `old_id`'s metadata.
- **Cascade**: `old_id` and transitive dependents flip to OUT (unless justified elsewhere).
- **Pub/sub**: If `pubsub.has_subscribers(db_path)` is true, a `compute_changes` + `publish` call fires after save, emitting truth-value deltas.
- **Cache**: `_network_cache[db_path]` is refreshed with the new network and mtime.
- **`retracted_at` timestamp**: `net.retract()` sets this on `old_id` (via the reason argument).
- **No justification, no defeater, no outlist edge added** — this is the distinguishing difference from `supersede()`. Dependents that depended on `old_id` as an antecedent will go OUT because their justifications are no longer satisfied, not because any new defeater was introduced.

## Error Handling

| Condition | Behavior |
|-----------|----------|
| `old_id == new_id` | `ValueError("A node cannot be marked as superseded by itself")` — raised *before* the write context opens. No side effects. |
| `old_id` not found | `KeyError(f"Node '{old_id}' not found")` — raised inside write context; `__exit__` sees `exc_type is not None` and **does not save**. |
| `new_id` not found | `KeyError(f"Replacement node '{new_id}' not found")` — same non-save path. |
| DB save fails | Propagates from `Storage.save()`; original metadata mutations are lost because the save didn't complete. |
| DB open fails | Propagates from `Storage(db_path)` constructor (e.g. `sqlite3.OperationalError`). |

Not handled: nothing catches exceptions from `net.retract()`. If retraction raises (e.g., corruption, most commonly a `KeyError` from a stale dependent reference), the whole operation aborts and nothing is persisted.

## Usage Patterns

```python
# Typical: caller has already created the replacement node
add_node(new_id="auth-login-v2", text="Use OAuth2 for auth")
mark_superseded(old_id="auth-login-v1", new_id="auth-login-v2")
# → auth-login-v1 is OUT with metadata marking the supersession
```

Common **follow-up** operations:
- `get_status()` — `superseded_count` counts nodes with `metadata["superseded_by"]`.
- `list_gated()` — explicitly **skips** superseded nodes so they don't appear as "gated by outlist."
- `migrate_retract_to_defeaters()` — converts these metadata-only retractions into graph-native `defeater` nodes on a per-node basis, so the retraction becomes reversible/inspectable through the TMS.

**Caller obligations**
- If you want the retraction to be reversible (restoring `old_id` if `new_id` goes OUT), use `supersede()` instead, not this function.
- If you want the replacement to be asserted IN, do that yourself (`assert_node(new_id)`) — this function never touches `new_id`.
- Don't rely on `changed` containing only dependents; it includes `old_id` too.
- If you plan to accept LLM/user proposals, prefer `propose_supersession()` → `accept_proposal()` which routes through `supersede_with_text()` (reversible) rather than calling this directly.

## Dependencies

- **`_with_network`** — context manager, handles load/save, cache, and pub/sub.
- **`Storage`, `Network`** — loaded indirectly by `_with_network`.
- **`Network.retract(node_id, reason)`** — performs the actual TMS cascade and `_retracted` flagging.
- **`pubsub`** — active only when subscribers exist.
- **`DEFAULT_DB`** — module constant `"reasons.db"`.

## Topics to Explore

- [function] `reasons/api.py:supersede` — The graph-native (reversible) alternative; uses outlist instead of `retract()`, so cascades behave differently when `new_id` goes OUT.
- [function] `reasons/api.py:supersede_with_text` — Higher-level variant that creates the replacement node and supersedes in one call; what most callers should use.
- [function] `reasons/api.py:migrate_retract_to_defeaters` — Shows why `mark_superseded` is considered legacy: converts its `retract_reason` metadata into graph-native defeater nodes.
- [function] `reasons/network.py:Network.retract` — The cascade engine used here; how `_retracted` flags, outlist evaluation, and dependents recomputation interact.
- [general] `proposals-accept-flow` — The modern mutation path (`propose_supersession` + `accept_proposal`) that intentionally avoids this string-metadata style.

## Beliefs

- `mark-superseded-is-irreversible` — `mark_superseded` uses `Network.retract()`, not an outlist justification, so the retraction cannot be undone by asserting `new_id`; only `supersede()` supports that.
- `mark-superseded-does-not-touch-new-id` — The function only mutates `old_id`'s metadata and truth value; `new_id`'s truth value and metadata are untouched, even if `new_id` is OUT.
- `mark-superseded-requires-both-nodes-to-preexist` — Unlike `supersede_with_text`, this function raises `KeyError` for the replacement if it has not been added to the network first.
- `mark-superseded-updates-metadata-before-cascade` — The `superseded_by` and `retract_reason` keys are written to `node.metadata` *before* `net.retract()` runs, so the cascade engine and later `get_status`/`list_gated` calls can read them.
- `mark-superseded-emits-pubsub-deltas` — Because it opens `_with_network(db_path, write=True)` with subscribers present, it captures `_before` truth values and publishes `pubsub.compute_changes` events on context exit.
