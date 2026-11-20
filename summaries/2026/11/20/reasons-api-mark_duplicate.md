---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: mark_duplicate in reasons/api.py

## `mark_duplicate` — `reasons/api.py`

### Purpose

Marks one belief as a duplicate of another and retracts the duplicate. It is a "record-and-retract" operation: it stamps metadata (`duplicate_of`, `retract_reason`) on the source node so downstream tools can see *why* the belief went OUT, then delegates to the network-level `retract()` which cascades truth-value changes through the justification graph.

It is the API-level counterpart to `mark_superseded` and is structurally near-identical to it — the only differences are the metadata keys written (`duplicate_of` vs `superseded_by`) and the human-readable `reason` text. It is also the *low-level* primitive; the automated dedup path (`apply_dedup_plan`, `deduplicate(auto=True)`) does more work (it rewrites dependents to point at the kept node via `_rewrite_dependents`) before retracting. `mark_duplicate` deliberately does **not** do that rewriting.

### Contract

**Preconditions**
- `source_id != canonical_id` (enforced by an explicit check).
- Both `source_id` and `canonical_id` must already exist in the network (enforced by `KeyError` lookups).
- Both IDs are expected to be **fully resolved** node IDs. Unlike `add_node`, `add_justification`, or `ensure_namespace`, `mark_duplicate` does **no namespace resolution** — it never calls `_resolve_namespace` and has no `namespace` parameter. If you pass a bare ID that needs prefixing, you'll get a `KeyError`.
- Backend is **SQLite only**. There is no `pg_conninfo` / `domain_id` parameter and no `_pg_dispatch` branch — unlike most other functions in this module.

**Postconditions**
- `source_node.metadata["duplicate_of"] == canonical_id`
- `source_node.metadata["retract_reason"] == f"Duplicate of {canonical_id}"`
- `source_id`'s truth value is `"OUT"` (and any dependents that lose their only support also go `OUT`).
- The database file has been saved and change events published (if there are subscribers).

**Invariants / assumptions**
- The `duplicate_of` relationship is *metadata only* — it does not participate in TMS propagation. Beliefs that depended on `source_id` are **not** rewired to depend on `canonical_id`; they simply go OUT if `source_id` was their only support.
- The function assumes `metadata` is a mutable `dict`; `node.metadata or {}` handles the `None`/empty case by creating a fresh dict, but the type is not enforced.

### Parameters

| Name | Type | Notes |
|---|---|---|
| `source_id` | `str` | Fully-resolved ID of the node to retract as a duplicate. Must exist. |
| `canonical_id` | `str` | Fully-resolved ID of the node to point at. Must exist. Must differ from `source_id`. |
| `db_path` | `str` | Path to the SQLite database. Defaults to `DEFAULT_DB = "reasons.db"`. |

Edge cases:
- Passing the same ID for both → `ValueError`.
- Passing a namespaced-looking ID (`ns:foo`) that isn't in the DB → `KeyError` (no auto-prefixing).
- `db_path` pointing at a non-existent file: `Storage(db_path)` is opened by `_with_network`, so this typically raises a `sqlite3`/`FileNotFoundError`-class error rather than producing a friendly message.

### Return Value

Returns a JSON-serializable dict:

```python
{
    "source_id": str,      # echoed back
    "canonical_id": str,   # echoed back
    "changed": list[str],  # node IDs whose truth value flipped during cascade
}
```

`changed` comes directly from `Network.retract(...)`. The caller must handle:
- Empty `changed` (e.g. `source_id` was already OUT — the metadata is still written but no propagation occurred).
- A large `changed` list if `source_id` had many dependents that lost their only justification.

### Algorithm

1. **Guard against self-reference.** If `source_id == canonical_id`, raise `ValueError`.
2. **Open a write context.** `with _with_network(db_path, write=True) as net:` loads a fresh `Network` (writes always load fresh, bypassing the read cache), and defers save/close/pubsub to `__exit__`.
3. **Validate existence.** `KeyError` if `source_id` or `canonical_id` is missing from `net.nodes`.
4. **Mutate metadata in place.** 
   ```python
   meta = node.metadata or {}
   meta["duplicate_of"] = canonical_id
   meta["retract_reason"] = f"Duplicate of {canonical_id}"
   node.metadata = meta
   ```
   Note the reassignment — if `node.metadata` was `None`/empty, `meta` is a brand-new dict; otherwise it's the same object mutated in place. Either way the result is the same. `net.retract` is *not* used for `duplicate_of` — that's purely metadata.
5. **Retract and cascade.** `net.retract(source_id, reason=reason)` returns the set/list of node IDs whose truth value changed. `reason` is passed through to the TMS log.
6. **Return** the three-field dict.
7. **On context exit** (success path): `_with_network.__exit__` calls `store.save(network)`, computes before/after truth-value deltas, and calls `pubsub.publish(...)`. It also refreshes the mtime-keyed cache.

### Side Effects

- **Database write**: entire network is persisted via `Storage.save()` at context exit.
- **Truth-value mutations**: `source_id` → OUT; transitive dependents → OUT.
- **Metadata mutations**: two keys written to `source_id`'s metadata.
- **TMS log**: `retract()` appends an entry with the duplicate reason.
- **Pub/Sub**: `pubsub.compute_changes` + `pubsub.publish` fire if `pubsub.has_subscribers(db_path)` — the `_with_network` write path checks this in `__enter__`.
- **Cache invalidation**: `_network_cache[db_path]` updated on exit.

### Error Handling

| Situation | Behavior |
|---|---|
| `source_id == canonical_id` | `ValueError` — raised before any DB I/O. |
| `source_id` not found | `KeyError("Node '...' not found")`. |
| `canonical_id` not found | `KeyError("Canonical node '...' not found")`. |
| DB file missing / corrupt | `sqlite3` / OSError propagates from `Storage(db_path)` inside `_with_network`. |
| Any exception inside the `with` | `_with_network.__exit__` skips save/pubsub (checks `exc_type is None`), still closes the store, does not update the cache. |

Note: if `canonical_id` lookup fails **after** `source_id` metadata was already... — actually no, the metadata write happens *after* both lookups, so a missing canonical ID leaves metadata untouched. Good ordering.

### Usage Patterns

```python
result = mark_duplicate("claim-42-old", "claim-42", db_path="reasons.db")
# → {"source_id": ..., "canonical_id": ..., "changed": ["claim-42-old", ...]}
```

Typical callers:
- A human-curated "this is a duplicate" decision where you don't need to rewire dependents.
- Feeding into `list_gated()` — the OUT source will appear as gated if its justification's outlist is IN.
- CLI/tooling that presents duplicates for review.

**Caller obligations / caveats:**
- If you *do* want dependents to survive by pointing at `canonical_id`, call `_rewrite_dependents(net, source_id, canonical_id)` or use `apply_dedup_plan` instead — `mark_duplicate` alone will cascade-retract them.
- IDs must already be namespaced; resolve with `_resolve_namespace` or prefix manually if needed.
- Do **not** call this expecting PostgreSQL support — there is no dispatch; it will try to open a SQLite file at `db_path`.
- Idempotency is *soft*: calling again on an already-OUT node still writes the metadata and calls `retract`, which should return an empty `changed` (already OUT). No exception.

### Dependencies

- `_with_network` — the load/operate/save/pubsub context manager.
- `DEFAULT_DB` — the module-level default path.
- `Network.retract(reason=...)` — cascade-retraction primitive.
- `Storage` / `pubsub` — indirectly, via `_with_network`.
- No imports from `postgres`/`pg.py`; this is a pure-SQLite path.

---

## Topics to Explore

- [function] `reasons/network.py:Network.retract` — Understand exactly how the retraction cascade and `changed` list are computed; this is what determines the meaning of the returned `changed`.
- [function] `reasons/api.py:mark_superseded` — The direct sibling; diffing the two shows the intentional metadata-key and reason-string split.
- [function] `reasons/api.py:_rewrite_dependents` — The mechanism `mark_duplicate` omits; explains why dependents go OUT here but survive in `apply_dedup_plan`.
- [function] `reasons/api.py:_with_network` — The write path, save, and pubsub semantics that give this function its persistence and side effects.
- [file] `tests/test_deduplication.py` — Shows expected behavior around duplicate discovery, `mark_duplicate`/`mark_superseded`, and the contrasting `_rewrite_dependents` flow.

## Beliefs

- `mark-duplicate-raises-on-self-reference` — Passing the same ID for `source_id` and `canonical_id` raises `ValueError`.
- `mark-duplicate-is-sqlite-only` — `mark_duplicate` has no `pg_conninfo`/`domain_id` parameters and never calls `_pg_dispatch`, unlike most sibling API functions.
- `mark-duplicate-does-not-rewrite-dependents` — The function records `duplicate_of` and retracts, but never rewires dependent justifications to point at `canonical_id` (contrast: `apply_dedup_plan` does).
- `mark-duplicate-produces-retraction-cascade` — Its returned `changed` list is the cascade of truth-value changes from `Network.retract(source_id)`, not just the source node.
- `mark-duplicate-stores-duplicate-of-metadata` — It writes `duplicate_of` and `retract_reason` into the source node's `metadata` dict before retracting.
