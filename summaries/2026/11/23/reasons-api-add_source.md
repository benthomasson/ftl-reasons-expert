---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: add_source in reasons/api.py

# `add_source` — Explanation

## 1. Purpose

`add_source` attaches a **provenance record** to an existing node. The RMS stores provenance in two places: node-level fields (`source`, `source_url`) and a separate `node_sources` table that can hold *multiple* sources per node with structured metadata (`source_type`, `source_ref`, `source_url`, `label`, `added_at`).

This function is the direct-write API for the second mechanism. It is invoked by callers (CLI wrappers, LangGraph tools, scripts) that want to record "this belief came from this file/URL/document" without going through the full `Network` load/save cycle.

Crucially, it **bypasses** `_with_network(...)`, `_network_cache`, and the `pubsub` event system entirely — sources do not affect truth values, so `_with_network`'s caching and change-detection are unnecessary here. It sits alongside `add_tags`, `remove_tags`, `get_tags` as a set of "flat table" operations that open sqlite directly.

## 2. Contract

**Preconditions**
- `node_id` must reference a row in the `nodes` table (checked at runtime via `SELECT id FROM nodes WHERE id = ?`).
- `pg_conninfo` must be falsy — otherwise `NotImplementedError` is raised.
- The `node_sources` table must exist with at least columns `(node_id, source_type, source_ref, source_url, label, added_at)`.
- `db_path` must point to a valid RMS sqlite database.

**Postconditions**
- A new row is inserted into `node_sources` with `added_at` set to the current UTC ISO 8601 timestamp (second precision).
- The transaction is committed before the connection closes.
- The next `show_node` / `export_network` call will include this source in the node's `sources` array (ordered by `rowid`).

**Invariants this function does *not* enforce**
- `source_type` is **not** validated against `SOURCE_TYPES` (`{"code", "document", "self-description", "derived"}`) — unlike `add_node`, which rejects unknown source types.
- Duplicate sources are **not** deduplicated — repeated calls create duplicate rows.
- Foreign-key enforcement in SQLite is off by default; if the node were deleted between the SELECT and INSERT (or if a caller deleted it concurrently), the FK would not fire.

## 3. Parameters

| Param | Type | Default | Notes |
|---|---|---|---|
| `node_id` | `str` | — (required) | Must be an existing node ID. Case-sensitive. |
| `source_ref` | `str` | — (required) | The primary reference (e.g. `repo:path/to/file.py:42`, `https://...`). Accepted as empty string — semantic validation is the caller's job. |
| `source_type` | `str` | `""` | Free-form here; the "canonical" set is `SOURCE_TYPES` but not enforced. |
| `source_url` | `str` | `""` | Optional link. |
| `label` | `str` | `""` | Optional human-readable tag. |
| `db_path` | `str` | `DEFAULT_DB` (`"reasons.db"`) | Any path accepted; opened lazily by `sqlite3.connect` (creates a new file if missing unless schema is absent, in which case the INSERT fails). |
| `pg_conninfo` | `Any` | `None` | Truthy → `NotImplementedError`. Empty string is falsy and would *not* trigger the guard (a caller-supplied empty string silently falls through to SQLite — a footgun). |
| `domain_id` | `Any` | `None` | Unused in the SQLite path. Reserved for the (unimplemented) PostgreSQL path. |

## 4. Return Value

On success:

```python
{"node_id": "<input node_id>", "source_id": <int>}
```

`source_id` is `cursor.lastrowid` — the SQLite implicit rowid (auto-incremented). The caller should persist this ID if they later need to identify the specific source entry (e.g., for deletion, though no `remove_source` exists yet). Note it is **not** the same number as `rowid` from a query — SQLite's `lastrowid` after INSERT is the rowid of the inserted row; if `node_sources` has an INTEGER PRIMARY KEY column, that's the value.

**Never returns a partial/failure dict** — errors raise.

## 5. Algorithm

1. **Reject PostgreSQL**: `if pg_conninfo: raise NotImplementedError(...)`.
2. **Open sqlite connection** to `db_path`.
3. **Verify node exists**: `SELECT id FROM nodes WHERE id = ?`. Fetch one row. If `None` → `KeyError`.
4. **Compute timestamp**: `datetime.now(timezone.utc).isoformat(timespec="seconds")` → e.g. `"2024-01-15T10:30:00+00:00"`.
5. **Insert**: `INSERT INTO node_sources (node_id, source_type, source_ref, source_url, label, added_at) VALUES (?, ?, ?, ?, ?, ?)` using positional parameters (safe from SQL injection).
6. **Commit** — persists the row.
7. **Return** `{"node_id": node_id, "source_id": cursor.lastrowid}`.
8. **Cleanup** in `finally`: `conn.close()`. If any step raised before `commit`, SQLite auto-rolls back the implicit transaction.

## 6. Side Effects

- **Database write**: one `INSERT` and one `COMMIT` against `node_sources`.
- **Schema dependency**: requires `node_sources` and `nodes` tables to exist. A fresh empty file passed as `db_path` will fail with `sqlite3.OperationalError: no such table: nodes`.
- **Cache invalidation side effect**: touching the file changes its mtime, so the next `_with_network(...)` read will miss the `_network_cache` and reload the network from disk. Sources are **not** part of `Network`, so this is a pure perf cost, not a correctness issue.
- **No pubsub event**: unlike `_with_network(write=True)` paths, this does not trigger `pubsub.publish`.
- **No Merkle/`updated_at` bump** on the node itself — adding provenance does not touch `node.updated_at`. So `check_stale`, `search --sort updated`, etc. will not reflect that a source was added.

## 7. Error Handling

| Exception | When |
|---|---|
| `NotImplementedError("add_source is not supported with PostgreSQL yet")` | `pg_conninfo` is truthy. |
| `KeyError(f"Node '{node_id}' not found")` | `SELECT id FROM nodes WHERE id = ?` returns no row. |
| `sqlite3.OperationalError` | `db_path` is missing/invalid; `nodes` or `node_sources` tables missing; disk full; etc. |
| `sqlite3.IntegrityError` | FK violation on `node_sources.node_id` if FK enforcement were enabled (it is not by default, so this is mostly theoretical). |
| `sqlite3.DatabaseError` | Corrupt database file. |

No exceptions are swallowed — everything propagates. The `finally: conn.close()` ensures the connection is released on all paths (including before `commit`; SQLite discards the uncommitted transaction).

## 8. Usage Patterns

**Typical call site** — after `add_node`, to attach provenance:

```python
add_node("my-belief", "some claim", source="src/foo.py", source_url="https://github.com/.../foo.py#L10")
add_source(
    "my-belief",
    source_ref="src/foo.py:10-25",
    source_type="code",
    source_url="https://github.com/.../foo.py#L10",
    label="original implementation",
)
```

**Caller obligations**
- Ensure the node exists first (`add_node` before `add_source`) — otherwise `KeyError`.
- If `source_type` matters, validate it against `SOURCE_TYPES` before calling (this function won't).
- Be aware of **non-idempotency**: calling twice with the same arguments creates two rows. Deduplicate at the caller or query `show_node` first.
- Do **not** mistake this for setting the node's `source` attribute — that's a field on `Node`, set via `add_node(source=..., source_url=...)`. `add_source` only writes to `node_sources`.
- Column gaps: `add_source` sets only 6 of the 9 `node_sources` columns. `source_hash`, `pinned_sha`, and `pinned_lines` are left `NULL`. Only `import_json` populates those. So if a caller later runs a hash/pin workflow, the source row will still be missing hash/pin info.

## 9. Dependencies

**Internal (same module)**
- `DEFAULT_DB = "reasons.db"` — fallback path constant.
- Implicitly: the schema of `nodes` and `node_sources` (defined in `reasons/storage.py` — `PROPOSALS_SCHEMA` and presumably a `SOURCES_SCHEMA` / DB init).

**External**
- `sqlite3` (stdlib) — connection, cursor, execute.
- `datetime.datetime` and `datetime.timezone` (stdlib) — UTC timestamp.

**Downstream consumers** (worth understanding for contract purposes):
- `show_node` — reads `node_sources` back into the `sources` field (with `source_hash`/`pinned_sha`/`pinned_lines` columns included).
- `export_network` — same read, ordered by `(node_id, rowid)`.
- `import_json` — the only other writer to `node_sources`, and it **does** populate all 9 columns.

---

## Topics to Explore

- [file] `reasons/storage.py` — Contains the actual `node_sources` table definition (columns, indexes, FK constraints). Confirm whether `PRAGMA foreign_keys=ON` is set anywhere and what `add_source` will fail on if it isn't.
- [function] `reasons/api.py:add_node` — Compare the alternative provenance API (`source`, `source_url`, `source_type` params) with `add_source`; note that `add_node` validates against `SOURCE_TYPES` while `add_source` does not.
- [function] `reasons/api.py:show_node` — Shows how sources are read back, including the extra `source_hash`, `pinned_sha`, `pinned_lines` columns that `add_source` leaves NULL.
- [function] `reasons/api.py:import_json` — The other writer to `node_sources`; a good reference for what a "complete" source row looks like and which columns `add_source` is skipping.
- [file] `tests/test_api.py` — Check what behaviors are actually tested for `add_source` (probably node-not-found, duplicate sources, and JSON serializability).

## Beliefs

- `add-source-is-not-idempotent` — Calling `add_source` twice with identical arguments inserts two rows in `node_sources`; there is no uniqueness constraint or deduplication.
- `add-source-rejects-postgres` — Any truthy `pg_conninfo` raises `NotImplementedError` before touching the database; PostgreSQL support is explicitly deferred.
- `add-source-requires-existing-node` — An unknown `node_id` raises `KeyError` before any insert; the existence check is a `SELECT id FROM nodes WHERE id = ?`.
- `add-source-skips-source-type-validation` — Unlike `add_node`, `add_source` accepts any `source_type` string; the `SOURCE_TYPES` allowlist is not enforced here.
- `add-source-writes-only-six-columns` — `add_source` populates `(node_id, source_type, source_ref, source_url, label, added_at)` and leaves `source_hash`, `pinned_sha`, `pinned_lines` as NULL; only `import_json` fills all nine.
- `add-source-bypasses-network-cache` — Because it opens sqlite directly rather than using `_with_network`, `add_source` does not update `_network_cache` or emit pubsub events; the next read reloads the network from disk due to the mtime change.
