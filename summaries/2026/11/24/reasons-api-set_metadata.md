---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: set_metadata in reasons/api.py

## Purpose

`set_metadata` is a small write-oriented function in the functional API layer. It sets a single metadata key on an existing node and persists the change through the same load/modify/save lifecycle used by the rest of `reasons/api.py`.

It exists so callers can annotate nodes without manually opening `Storage`, loading a `Network`, saving it, and closing it. It also provides the standard SQLite/PostgreSQL dispatch pattern used across the API: use SQLite when `pg_conninfo` is falsy, otherwise delegate to `PgApi`.

## Contract

### Preconditions

- On the SQLite path, `node_id` must already exist in the loaded network.
- `db_path` must point to a usable SQLite database path understood by `Storage`.
- `node.metadata` should be either `None` or a mutable dict-like object. The function assumes `node.metadata or {}` yields a mapping that supports `meta[key] = value`.
- `value` must be acceptable to the storage backend’s serialization. This is not validated here.
- `key` is expected to be a string, though this is only enforced by the type annotation and not by runtime checks.
- When `pg_conninfo` is truthy, PostgreSQL-specific preconditions are delegated to `PgApi.set_metadata`.

### Postconditions

On the SQLite path, after a successful call:

- `net.nodes[node_id].metadata[key] == value`
- `net.nodes[node_id].updated_at` is set to the current UTC time as an ISO-8601 string with second-level precision.
- The modified network is saved by `_with_network(db_path, write=True)` when the context exits normally.
- The network cache is updated with the new file mtime and network object after a successful write.
- The function returns `{"node_id": node_id, "key": key}`.

On the PostgreSQL path:

- The call is delegated to `PgApi.set_metadata`.
- The return value is whatever `PgApi.set_metadata` returns; it is not constructed locally.

### Invariants

- `set_metadata` does not change `truth_value`.
- It does not add, remove, or modify justifications.
- It does not recompute dependencies or propagate truth values.
- It only mutates the targeted node’s metadata and `updated_at`.
- It does not resolve namespaces. The caller must pass the exact stored node ID.

## Parameters

### `node_id: str`

The node whose metadata should be updated.

Important edge case: unlike `add_node`, this function has no `namespace` parameter and does not call `_resolve_namespace`. If the stored node is `agent:belief-1`, the caller must pass `agent:belief-1`, not `belief-1`.

If the node does not exist on the SQLite path, the function raises:

```python
KeyError(f"Node '{node_id}' not found")
```

### `key: str`

The metadata key to set.

There is no validation against reserved keys, internal keys, or keys starting with `_`. Note that `export_network` filters metadata keys starting with `_` out of exported metadata, but `set_metadata` itself will happily write them. This can be surprising if a caller later expects the key to appear in exported network JSON.

### `value`

The metadata value to assign. The parameter is intentionally unannotated, so any Python value can be passed.

There is no type checking and no serialization check here. If the value is not serializable by `Storage.save` on the SQLite path, the in-memory mutation succeeds and the failure will occur later when the context manager saves the network. On the PostgreSQL path, backend-specific validation applies.

The value is not copied.

### `db_path: str = DEFAULT_DB`

SQLite database path. It defaults to `"reasons.db"`.

This parameter is ignored when `pg_conninfo` is truthy.

### `pg_conninfo=None`

If truthy, the function dispatches to the PostgreSQL backend via `_pg_dispatch` and `PgApi.set_metadata`.

Because this is a truthiness check, values like `None`, `False`, and `""` select the SQLite path.

### `domain_id=None`

Passed through to `_pg_dispatch` and therefore to `PgApi` when using PostgreSQL. Ignored on the SQLite path.

## Return Value

On the SQLite path, the function returns:

```python
{"node_id": node_id, "key": key}
```

It does not return the value, the full metadata dict, the updated timestamp, or the truth value.

On the PostgreSQL path, it returns the result of:

```python
PgApi(...).set_metadata(node_id=node_id, key=key, value=value)
```

The local function does not normalize or reshape that return value.

Callers must be prepared for:

- `KeyError` if the node does not exist on the SQLite path.
- Backend-specific exceptions from `PgApi`.
- Storage/save errors if metadata serialization fails.

## Algorithm

### SQLite path

1. `pg_conninfo` is falsy, so the function enters the SQLite branch.

2. It calls:

   ```python
   with _with_network(db_path, write=True) as net:
   ```

   Because `write=True`, `_with_network` does not reuse the read cache. It creates a fresh `Storage(db_path)`, loads the network via `store.load()`, and returns the network from `__enter__`.

   If there are pubsub subscribers, `_with_network` also snapshots node truth values before the write so it can later compute truth-value changes. Since `set_metadata` does not change truth values, this snapshot should normally produce no truth-value events.

3. It checks:

   ```python
   if node_id not in net.nodes:
       raise KeyError(...)
   ```

   If the node is missing, the exception propagates and the write context does not save.

4. It retrieves the node:

   ```python
   node = net.nodes[node_id]
   ```

5. It obtains a mutable metadata mapping:

   ```python
   meta = node.metadata or {}
   ```

   If metadata is truthy, this uses the existing dict. If metadata is `None` or an empty dict, this creates a new empty dict.

6. It assigns the key:

   ```python
   meta[key] = value
   ```

7. It assigns the mapping back:

   ```python
   node.metadata = meta
   ```

8. It updates the node timestamp:

   ```python
   node.updated_at = datetime.now(timezone.utc).isoformat(timespec="seconds")
   ```

   This happens even if the metadata value is unchanged.

9. It returns the minimal result dict from inside the `with` block. The context manager’s `__exit__` runs before the caller receives the return value.

10. On normal exit, `_with_network.__exit__` saves the entire network:

    ```python
    self.store.save(self.network)
    ```

    It then closes storage and, if the write succeeded, updates the network cache with the new mtime and network object.

### PostgreSQL path

1. `pg_conninfo` is truthy.

2. The function calls `_pg_dispatch(pg_conninfo, domain_id, "set_metadata", node_id=..., key=..., value=...)`.

3. `_pg_dispatch` imports `PgApi`, opens it as a context manager with `pg_conninfo` and `domain_id`, and calls its `set_metadata` method.

4. The result of that backend call is returned directly.

## Side Effects

- Mutates the targeted node’s `metadata` dict.
- Overwrites the targeted key if it already exists.
- Updates the targeted node’s `updated_at` timestamp.
- Persists the entire network to SQLite through `_with_network(write=True)`.
- Updates `_network_cache[db_path]` after a successful SQLite write.
- May publish pubsub events through `_with_network` if there are subscribers, though `set_metadata` itself does not alter truth values.
- On the PostgreSQL path, performs a remote database write through `PgApi`.

## Error Handling

### Missing node

On the SQLite path:

```python
raise KeyError(f"Node '{node_id}' not found")
```

This is not swallowed. Because the exception occurs inside the `with` block, `_with_network` skips the save path.

### Invalid metadata shape

The function assumes `node.metadata` is `None` or dict-like. If it is set to some other truthy object, `meta[key] = value` may raise `TypeError`.

### Non-serializable value

No validation is done before assignment. If `Storage.save` cannot serialize the value, the save step inside `_with_network.__exit__` may raise. The caller receives that exception rather than the local return dict.

### Storage/sqlite errors

Errors from `Storage.load`, `Storage.save`, sqlite connection handling, file permissions, or database schema issues are not caught by `set_metadata`.

### PostgreSQL errors

Errors from `_pg_dispatch`, `PgApi` construction, connection setup, or `PgApi.set_metadata` propagate to the caller.

## Usage Patterns

Typical calls pass an explicit `db_path`:

```python
set_metadata("belief-123", "review_result", "pass", db_path="reasons.db")
```

For PostgreSQL, callers pass `pg_conninfo` and optionally `domain_id`:

```python
set_metadata(
    "belief-123",
    "review_result",
    "pass",
    pg_conninfo=conninfo,
    domain_id="my-domain",
)
```

In this file, `set_metadata` is used to stamp operational metadata such as:

- `defeat_reason_type` during defeater classification.
- `repair_action` during premise repair.
- `repair_action = "rewritten"` on a newly superseding node.
- `repair_action = "retracted"` on a retracted node.

Caller obligations:

- Pass the exact stored node ID.
- Use a serializable metadata value.
- Handle `KeyError` if the node may not exist.
- Avoid using `set_metadata` for metadata that must participate in truth-value propagation; this function does not trigger recomputation.
- Be aware that setting metadata like `access_tags` is possible but not access-controlled or validated here.

## Dependencies

- `_with_network` — SQLite load/modify/save context manager.
- `Storage` — database persistence layer.
- `Network` / node model — provides `net.nodes` and `node.metadata`.
- `pubsub` — indirectly used by `_with_network` for write notifications.
- `_pg_dispatch` — PostgreSQL dispatch helper.
- `PgApi` — imported lazily by `_pg_dispatch`.
- `datetime`, `timezone` — timestamp generation.
- `DEFAULT_DB` — default SQLite path.

## Topics to Explore

- [file] `reasons/api.py` — See how the rest of the functional API uses the same SQLite/PostgreSQL dispatch and `_with_network` pattern.
- [function] `reasons/api.py:_with_network` — Understand load/modify/save behavior, write caching, pubsub change detection, and when persistence actually occurs.
- [file] `reasons/storage.py` — Explore how `Storage.save` serializes metadata and what values can cause save-time failures.
- [file] `reasons/pg.py` — Inspect `PgApi.set_metadata` to confirm its return value, validation, and error behavior.
- [general] `metadata-serialization-and-reserved-keys` — Investigate how keys like `_retracted`, `access_tags`, and other metadata fields are treated by export, storage, and access control.

## Beliefs

- `set-metadata-requires-existing-node` — On the SQLite path, `set_metadata` raises `KeyError` when `node_id` is absent from the loaded network.
- `set-metadata-delegates-postgres` — When `pg_conninfo` is truthy, `set_metadata` calls `PgApi.set_metadata` through `_pg_dispatch` and ignores `db_path`.
- `set-metadata-updates-updated-at` — A successful SQLite call assigns `node.updated_at` a UTC ISO-8601 timestamp with seconds precision.
- `set-metadata-persists-through-with-network` — Persistence is performed by `_with_network(write=True)`, which saves the entire network when the context exits normally.
- `set-metadata-does-not-change-truth-values` — The function changes only metadata and `updated_at`; justifications, dependents, and `truth_value` are untouched.
