---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: add_tags in reasons/api.py

# `add_tags` — `reasons/api.py:add_tags`

## Purpose

Appends one or more string tags to a node's tag list, storing them in the `node_tags` side table. Tags are metadata attached to nodes that are used for access-control (`access_tags` in metadata) and for search filtering (`search(tag={...})`, `list_nodes(tag=...)`).

The function exists as a thin, side-effect-only API surface: it does **not** touch the belief network's truth-value semantics. Tagging a node is orthogonal to belief propagation, so it deliberately bypasses the `_with_network` (load/save/`Storage`) machinery used by most other functions in this module and talks to SQLite directly.

## Contract

**Preconditions**
- `db_path` points to an existing RMS database file with a `nodes` table and a `node_tags` table. (`add_tags` does **not** create the schema; that is done by `init_db` / `Storage`.)
- The `node_tags` table must have a uniqueness constraint on `(node_id, tag)` (a `PRIMARY KEY` or `UNIQUE`) for `INSERT OR IGNORE` to be idempotent. This is **not** enforced by this function — see "Assumptions" below.
- `pg_conninfo` must be falsy (the function raises `NotImplementedError` otherwise).

**Postconditions**
- Every `tag` in `tags` that was not already present for `node_id` has been inserted.
- The database has been committed.
- The returned `tags` list is the **complete, sorted** tag set for the node (existing ∪ new), not just the newly-added tags.
- The connection to the database is closed (via `finally`).

**Invariants preserved**
- No truth values (`IN` / `OUT`) change; no justifications are added or removed.
- `_network_cache` is **not** invalidated. This is safe only because tags are not part of the cached `Network` object — code paths that need tags (e.g. `show_node`, `export_network`, `_get_node_tags`) always re-open a fresh `sqlite3` connection to the `node_tags` table rather than reading from `Network`.

## Parameters

| Param | Type | Notes |
|---|---|---|
| `node_id` | `str` | Must correspond to an existing row in `nodes.id`. Composite namespaced IDs (`ns:node`) are accepted because the check is a plain WHERE equality against `nodes.id`. |
| `tags` | `list[str]` | Each element is inserted individually. Empty list → no inserts, still returns current tags. Duplicates within the list are harmlessly absorbed by `INSERT OR IGNORE` (if the unique constraint exists). No format validation is performed — empty strings, very long strings, and strings with `:` or whitespace are all accepted and stored verbatim. |
| `db_path` | `str` | Defaults to `DEFAULT_DB = "reasons.db"`. Passed directly to `sqlite3.connect` — relative paths resolve against the current working directory, not any project root. |
| `pg_conninfo` | `Any` | If truthy, the function immediately raises `NotImplementedError`. Exists for API uniformity with other functions in this module that dispatch to a `PgApi` backend. |
| `domain_id` | `Any` | Accepted but **unused**. Passed in for signature uniformity with the PostgreSQL dispatch path taken by sibling functions. |

## Return Value

Returns a dict:

```python
{"node_id": <str>, "tags": <list[str]>}
```

- `node_id` echoes the input unchanged (not normalised).
- `tags` is the full tag set for the node, sorted lexicographically by SQL `ORDER BY tag` (SQLite's default collation is byte-wise, so ordering is deterministic but case-sensitive).

Callers must handle:
- A successful add — the list is authoritative current state.
- `KeyError` for a nonexistent node (see Error Handling).

## Algorithm

1. **Reject PostgreSQL.** If `pg_conninfo` is truthy → `raise NotImplementedError(...)`. There is no dispatch to `_pg_dispatch` here because the pg backend hasn't implemented tag writes.
2. **Open a direct SQLite connection** to `db_path`. This is a plain `sqlite3.connect`, **not** `Storage(db_path)` — so no `Storage`-level PRAGMAs (WAL, foreign keys, tag-table bootstrap) are applied. Same caveat as other direct-connection helpers in this file.
3. **Existence check.** `SELECT id FROM nodes WHERE id = ?` with the node_id bound as a parameter. If `fetchone()` returns `None` → `raise KeyError(...)`.
4. **Insert loop.** For each tag, execute:
   ```sql
   INSERT OR IGNORE INTO node_tags (node_id, tag) VALUES (?, ?)
   ```
   `OR IGNORE` swallows constraint violations (typically PK/UNIQUE conflicts from re-adding an existing tag, but also FK failures and other constraint failures if the schema enforces them).
5. **Commit.** Single transaction covering all inserts.
6. **Re-read.** Query all tags for the node in sorted order and project the first column into a list. This is the "canonical" post-state to return — including tags that were present before this call.
7. **Close** the connection in `finally` (runs on both success and exception).

## Side Effects

- **Database writes**: up to `len(tags)` rows in `node_tags`.
- **Database commit**: fires inside `try`, so on success subsequent reads see the new state.
- **Connection lifecycle**: opens and closes an `sqlite3.Connection`.
- **No in-memory state mutation**: `_network_cache` is untouched; `Network` objects are never loaded. This is intentional and correct *only* because the tag table is authoritative and always re-read from SQLite.
- **No pubsub notifications**: `pubsub.publish` is not called (unlike write-paths through `_with_network`). Subscribers to `db_path` will not see tag updates.

## Error Handling

**Raised (propagated to caller):**

| Exception | When |
|---|---|
| `NotImplementedError` | `pg_conninfo` is truthy. |
| `KeyError` | No row in `nodes` with the given `id`. |
| `sqlite3.OperationalError` | Database file missing/unwritable, `nodes` or `node_tags` table absent, SQL syntax error (e.g. schema drift). |
| `sqlite3.DatabaseError` | Corruption or similar low-level failures. |
| `sqlite3.IntegrityError` | Only possible if a **non-`OR IGNORE`** constraint fires — in practice not reachable because `INSERT OR IGNORE` downgrades constraint violations to no-ops. FK violations are also absorbed. |
| Any `sqlite3` exception during `commit()` | Propagation stops before re-read; the connection still closes. |

**Swallowed:**
- Duplicate-tag insertions (by design, via `OR IGNORE`).
- Foreign-key violations on `node_id` (also via `OR IGNORE`), *if* `PRAGMA foreign_keys=ON` was set on the connection — which it isn't here. So this is a theoretical concern only if the caller has set a global PRAGMA elsewhere (they haven't, because PRAGMAs are per-connection).

**Not returned:** this function never returns an error dict on failure — it raises. That's distinct from most of the read-API functions in this module, which raise `KeyError`/`PermissionError` but return dicts on success.

## Usage Patterns

```python
# Typical
add_tags("belief-42", ["reviewed", "topic:networking"])
# → {"node_id": "belief-42", "tags": ["reviewed", "topic:networking"]}

# Idempotent re-add
add_tags("belief-42", ["reviewed"])
# → {"node_id": "belief-42", "tags": ["reviewed", "topic:networking"]}

# Batch add
add_tags("belief-42", ["pinned", "verified"])
```

**Caller obligations:**
- Use it *after* `add_node` / `import_*` — the node must already exist in `nodes`.
- Do not expect it to validate tags; downstream consumers like `_filter_by_tags` do a `contains` match on the stored strings, so tag vocabulary/prefix conventions (`topic:`, `status:`) must be maintained by the caller.
- Do not rely on the network cache invalidation — it is not triggered here. If you subsequently call `search(tag=...)` or `list_nodes(tag=...)`, those paths re-read from `node_tags` so they will see fresh tags. But if you have any custom code caching tags in-process, you must catch that yourself.

**Access-control note:** `access_tags` — the field that `_is_visible` checks for filtering in `show_node`/`search` — is stored in `node.metadata["access_tags"]`, **not** in `node_tags`. Calling `add_tags(node, ["access_tags:secret"])` does not gate visibility. Do not confuse the two mechanisms.

## Dependencies

- `sqlite3` (stdlib).
- `DEFAULT_DB` constant from the same module (default `"reasons.db"`).
- Implicit schema from `Storage` / `init_db`: the `nodes(id)` and `node_tags(node_id, tag)` tables.
- Nothing else — no imports from `.network`, `.storage`, `.pubsub`, or `.metadata`, unlike sibling functions.

## Assumptions Not Enforced by Types

1. **`node_tags` has a `(node_id, tag)` PRIMARY KEY or UNIQUE constraint.** Without it, `INSERT OR IGNORE` never fires on duplicates and re-calling `add_tags` with the same tag will insert duplicate rows; the returned `tags` list would then contain duplicate strings.
2. **`node_tags.node_id` is a real FK to `nodes.id`.** But `PRAGMA foreign_keys` is OFF by default on a fresh `sqlite3.connect`, and this function does not enable it. So the FK, if declared, is not actually enforced here — the `SELECT id FROM nodes` pre-check is the only protection.
3. **Tags are not expected to be UNIQUE across nodes.** No cross-node tag registry exists; each node maintains its own list.
4. **Tags don't affect truth propagation.** The TMS engine (`Network`, `Justification`, `retract`) never inspects `node_tags`, so tagging is purely orthogonal metadata.
5. **`domain_id` is meaningless on the SQLite backend.** It is only consumed by `_pg_dispatch` in other functions; here it is a placeholder for future parity.

---

## Topics to Explore

- [file] `reasons/storage.py` — Defines the `node_tags` schema (`PROPOSALS_SCHEMA` and the `Storage` DDL), including whether `(node_id, tag)` has a UNIQUE constraint — this determines whether `INSERT OR IGNORE` is actually idempotent.
- [function] `reasons/api.py:remove_tags` — Symmetric removal API; compare its return shape (`removed` field) with `add_tags` to understand the tag-lifecycle contract.
- [function] `reasons/api.py:_filter_by_tags` — Shows how the strings deposited by `add_tags` are consumed at query time (prefix expansion for keys other than `"tag"`, AND-of-OR semantics).
- [function] `reasons/api.py:show_node` — The primary read-path that surfaces `node_tags` back to callers alongside node metadata; illustrates why tag reads bypass the network cache.
- [general] `node-tags-vs-access-tags` — Two parallel tag systems (`node_tags` table vs `metadata["access_tags"]`); understanding the distinction is required before adding auth/visibility logic.

## Beliefs

- `add_tags-bypasses-network-cache` — `add_tags` writes directly via `sqlite3.connect`, never opening `_with_network` and never invalidating `_network_cache`.
- `add_tags-requires-unique-constraint` — Idempotent behaviour of `INSERT OR IGNORE` depends on a UNIQUE/PRIMARY KEY constraint on `(node_id, tag)` that this function does not create or verify.
- `add_tags-raises-keyerror-for-missing-node` — A nonexistent `node_id` raises `KeyError`; the function never returns an error-shaped dict.
- `add_tags-returns-full-tag-list` — The return value contains all tags currently stored for the node (sorted), not only the tags passed in.
- `add_tags-rejects-postgres` — Setting `pg_conninfo` triggers `NotImplementedError`; there is no PostgreSQL dispatch path for tag writes.
