---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: supersede_with_text in reasons/api.py

## `supersede_with_text` — `reasons/api.py`

### Purpose

A convenience API that combines two operations in one call: **create a successor node** with new text, then **mark the old node as superseded** by the new one. It exists because callers (CLI, repair flows, proposal acceptance) frequently want "replace this belief's text with a better version" as a single atomic step, without manually creating a node and then wiring up the supersession relationship.

It is the *successor-with-content* counterpart to `supersede(old_id, new_id)`, which expects the successor node to already exist.

### Contract

**Preconditions**
- `old_id` must exist in the loaded network.
- Any explicitly supplied `new_id` must not collide with an existing node (this is *not* checked — see "Assumptions" below).

**Postconditions** (on success)
- A new premise node exists at `new_id` (or auto-generated ID) with `text=new_text`.
- The new node inherits `source`, `source_url`, and `metadata.access_tags` from the old node. No other metadata is copied.
- `net.supersede(old_id, new_id, transitive=transitive)` has been invoked — the old node goes OUT via the outlist mechanism once the new node is IN.
- `_with_network` saves the network to disk and publishes change events (if subscribers exist).

**Invariants preserved**
- Every node in the network has a non-empty `id` and `text` (assuming `net.add_node` enforces this).
- Access-control lineage: if the old node was gated behind `access_tags`, the successor inherits those tags — otherwise a caller could launder access restrictions by "upgrading" restricted beliefs into unrestricted ones.

### Parameters

| Parameter | Type | Notes |
|---|---|---|
| `old_id` | `str` | Node being superseded. Must exist; otherwise `KeyError`. |
| `new_text` | `str` | Body of the successor. No length/emptiness validation — an empty string will be accepted. |
| `new_id` | `str \| None` | Optional successor ID. Falsy (`None`/`""`) triggers auto-generation. Not collision-checked when supplied. |
| `db_path` | `str` | SQLite path, defaults to `"reasons.db"`. Passed directly to `_with_network`/`Storage`. |
| `transitive` | `bool` | Forwarded to `net.supersede`. When `True`, supersession "defeats all ancestors" in the chain so only the newest version of a lineage is IN. |

Notably, there is **no `pg_conninfo`/`domain_id`** parameter — unlike nearly every other function in `api.py`, this one is SQLite-only and cannot dispatch to the PostgreSQL backend.

### Return Value

Returns whatever `net.supersede` returns (whose own docstring promises):

```python
{"old_id": str, "new_id": str, "changed": list[str]}
```

`changed` is the list of node IDs whose truth values changed during propagation — this may include nodes the caller did not expect (dependents of the old node that cascade OUT, or outlist-gated nodes that cascade back IN because the successor shifted the outlist landscape). Callers must tolerate unexpected entries here.

Because the return is `net.supersede`'s dict directly, `new_id` in the returned dict is the *actual* successor ID — including any auto-generated value. Callers must read it from the return, not from the argument.

### Algorithm

1. **Open the network for write** via `_with_network(db_path, write=True)`. This bypasses the read cache, loads from storage, and — on clean context exit — re-saves and publishes change events.
2. **Validate the old node exists.** If not, raise `KeyError` — which aborts before any mutation.
3. **Resolve `new_id`.** If falsy:
   - Try `f"{old_id}-v2"`. If taken, increment to `-v3`, `-v4`, … until free.
   - Never try `-v1` — the convention implies the original was "v1", so the first successor is "v2".
4. **Snapshot inheritance fields** from the old node: `source`, `source_url`, and `metadata["access_tags"]` (wrapped in `{"access_tags": ...}` only if present — otherwise empty metadata dict).
5. **Add the successor** as a bare premise (no justifications) with the resolved ID, `new_text`, and inherited provenance.
6. **Call `net.supersede(old_id, new_id, transitive=transitive)`.** By the docstring of `supersede`, this leaves the old node OUT while the new node is IN via `_outlist` mechanics on the old node's justifications.
7. **Return the supersede result dict**; context manager saves on the way out.

### Side Effects

- **Writes to the DB.** New node row + revised justifications on `old_id` (via the outlist that `net.supersede` installs).
- **Cache invalidation.** `_with_network` updates `_network_cache[db_path]` with a fresh `mtime` on exit.
- **Pub/sub notification.** If `pubsub.has_subscribers(db_path)`, truth-value diffs are computed and published.
- **No filesystem writes outside the DB.** Unlike `add_source` or import functions, no extra files are created.

### Error Handling

- `KeyError(f"Node '{old_id}' not found")` — old node missing. Raised *before* any mutation, so the DB is untouched.
- **No explicit error on `new_id` collision.** When `new_id` is explicitly provided and already in the network, behavior depends on `Network.add_node` — either it overwrites silently, raises its own error, or misbehaves. This is a latent bug surface; callers who pass `new_id` must pre-check.
- **`Storage` / `sqlite3` errors** propagate from `_with_network` (`Storage(db_path)` constructor, `.load()`, `.save()`).
- **Exceptions inside the `with` block suppress the save** — `_Ctx.__exit__` only saves when `exc_type is None`. Any `KeyError` from step 2 therefore leaves the on-disk state unchanged.

### Usage Patterns

Callers used in this file:

- **`repair_premises`** — when an LLM proposes a `rewrite` action, calls `supersede_with_text(nid, r["corrected_text"], db_path=db_path)`. The returned `new_id` is then tagged with `metadata["repair_action"] = "rewritten"`.
- **`accept_proposal`** — on a pending `supersede` proposal, calls `supersede_with_text(target_id, proposed_text, new_id=new_id or None, db_path=db_path)`. Prior re-validation in `accept_proposal` guarantees `new_id` is free, compensating for the missing collision check here.

Obligations for new callers:
- Do not assume `new_id` == argument; read it from the result.
- Validate `new_id` collisions yourself if you pass one explicitly.
- If you care about non-access metadata surviving supersession (`example`, `source_type`, `accepted_pr`, `challenges`, etc.), re-apply it manually — it is **not** copied.

### Dependencies

- `reasons/api.py:_with_network` — the load/save/pubsub context manager.
- `reasons/network.py:Network.add_node` — node construction.
- `reasons/network.py:Network.supersede` — outlist supersession installation.
- `reasons/storage.py:Storage` — SQLite persistence (loaded via `_with_network`).
- `reasons/pubsub.py` — change event publishing.

### Assumptions Not Enforced by Types

1. **`new_id` collision is unchecked when caller supplies it.** Types say `str | None`, not "free ID." `accept_proposal` pre-checks; other callers must too.
2. **Only `access_tags` is inherited from metadata.** The function destructures one key and silently drops the rest. Types allow arbitrary `dict` metadata; the function assumes `access_tags` is the only metadata field with access-control implications.
3. **`old_node.source` and `old_node.source_url` are `str`-compatible**, so they can be fed directly to `add_node(source=..., source_url=...)`. If a future `Node` allowed `None`, this would break.
4. **`net.supersede` is assumed to be atomic-with-respect-to-calling-context** — it mutates in-memory state and returns a dict; if it raised mid-way, the save wouldn't run (safe), but the in-memory cache would still be updated by `_Ctx.__exit__` (because `not (write and exc_type is not None)` — this path only fires when write is False or no exception). Actually, on exception with `write=True`, the cache is **not** updated, which is correct.
5. **No namespace awareness.** Unlike `add_node`, this function does not take a `namespace` parameter and does not attempt `_resolve_namespace` — so `old_id` with a `:` in it is used verbatim and the successor is a bare ID, not namespaced.
6. **`new_text` may be empty** — nothing rejects it, so callers can accidentally poison the belief base with a blank successor.

---

## Topics to Explore

- [function] `reasons/network.py:Network.supersede` — Understand exactly what outlist mechanism "old goes OUT when new is IN" relies on, and how `transitive=True` behaves.
- [function] `reasons/api.py:_with_network` — The load/save/pubsub/cache context manager that underpins every mutation in this module.
- [function] `reasons/api.py:accept_proposal` — The primary safety-critical caller; shows how re-validation compensates for `supersede_with_text`'s missing collision check.
- [file] `reasons/pubsub.py` — How truth-value-change events are diffed and delivered after each write.
- [general] `tms-outlist-supersession` — The Truth Maintenance System pattern of reversible supersession via outlists (vs. destructive `retract_node`).

## Beliefs

- `supersede-with-text-inherits-access-tags-only` — The successor copies `metadata.access_tags`, `source`, and `source_url` from the old node; all other metadata is dropped.
- `supersede-with-text-is-sqlite-only` — The function has no `pg_conninfo`/`domain_id` parameters, so it cannot dispatch to the PostgreSQL backend like most other functions in `api.py`.
- `supersede-with-text-skips-v1-naming` — Auto-generated successor IDs start at `{old_id}-v2`, never `{old_id}-v1`, and increment infinitely on collision.
- `supersede-with-text-requires-old-id-exists` — A missing `old_id` raises `KeyError` before any mutation, so on-disk state is untouched on that failure.
- `supersede-with-text-does-not-validate-new-id-collisions` — When the caller supplies `new_id`, no existence check is performed; collision safety is delegated to callers such as `accept_proposal`.
