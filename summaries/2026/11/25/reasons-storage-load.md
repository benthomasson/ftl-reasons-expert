---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: load in reasons/storage.py

# `Storage.load` — Explanation

## Purpose

`Storage.load` reconstructs an in-memory `Network` from the SQLite database owned by a `Storage` instance. It is the read-side counterpart to `Storage.save`: it hydrates nodes, per-node justifications, nogoods, network metadata, repos, and the propagation log.

A key design choice is that `load` **does not replay graph mutations through `Network.add_node`**. Instead, it constructs `Node` objects directly and assigns them into `network.nodes`, then explicitly calls `network._rebuild_dependents()`. This preserves the exact persisted state—IDs, truth values, timestamps, justification ordering, etc.—without triggering validation, logging, or other side effects that `add_node` might perform.

The function also contains backward-compatibility logic for older SQLite schemas by inspecting available columns with `PRAGMA table_info(...)` and substituting literals like `NULL` or `''` in the `SELECT`.

## Contract

### Preconditions

- `self.conn` is an open `sqlite3.Connection` to a database that was initialized via `Storage.__init__` / `_init_schema`.
- Core tables such as `nodes`, `justifications`, `nogoods`, and `propagation_log` exist. `_init_schema` normally ensures this.
- JSON-bearing columns (`antecedents_json`, `outlist_json`, `metadata_json`, `nodes_json`) contain valid JSON with shapes expected by `Justification`, `Node`, and `Nogood`.
- Justification rows normally reference existing node IDs. This is not validated during load.

### Postconditions

- Returns a new `Network` instance.
- `network.nodes` contains all loaded nodes, each with its justifications attached.
- `network.nogoods`, `network.meta`, `network.repos`, and `network.log` are populated from the corresponding tables where available.
- `network._rebuild_dependents()` has been called, so the dependent index is rebuilt from the loaded justifications.
- No database writes or commits occur.

### Invariants

- Node IDs are unique because `nodes.id` is a primary key.
- Per-node justification order is preserved because justifications are selected `ORDER BY rowid` and appended in that order.
- `next_nogood_id` is intentionally skipped when loading `network_meta`; it is not inserted into `network.meta`.
- Orphan justifications—rows whose `node_id` has no matching node—are silently kept in a temporary map but never attached to any node.

### Assumptions Not Enforced by the Type System

- All non-checked columns exist. `load` checks for `source_url`, timestamps, `supporting_justification`, `text_hash`, and `content_hash`, but still assumes columns like `source_hash`, `date`, and `metadata_json` are present.
- JSON values decode into the expected Python types. For example, `metadata_json` is assumed to decode to a dict, and `antecedents_json` to a list.
- The `nodes` table schema has the expected column names/order for the chosen `SELECT` branch.
- `network_meta` and `repos` are optional/best-effort; failures reading them are swallowed.
- Normalized tables `node_tags` and `node_sources` are **not** merged into the returned `Network`; only `nodes.metadata_json` is read.

## Parameters

- `self`: A `Storage` instance.
  - Expected to have an active `sqlite3.Connection` in `self.conn`.
  - Edge cases: closed connection, missing tables, old schema, malformed JSON, or orphan justifications.

There are no other parameters.

## Return Value

Returns a fully instantiated `Network`.

The caller must be prepared for exceptions if the database is malformed or inconsistent:
- `sqlite3.Error` / `sqlite3.OperationalError` for missing tables or columns not handled by the migration checks.
- `json.JSONDecodeError` for invalid JSON in serialized fields.
- `ValueError` if a `SELECT` unexpectedly returns the wrong number of columns for tuple unpacking.

If `network_meta` or `repos` cannot be read, the function still returns a `Network`; those errors are swallowed.

## Algorithm

1. Create an empty `Network`.

2. Inspect the `nodes` table columns:
   ```python
   cols = [c[1] for c in self.conn.execute("PRAGMA table_info(nodes)").fetchall()]
   ```
   Determine flags for:
   - `source_url`
   - `created_at` / timestamp columns
   - `supporting_justification`
   - `text_hash`

3. Execute a `SELECT` that always returns 15 columns:
   ```
   id, text, truth_value, supporting_justification,
   source, source_url, source_hash, date, metadata_json,
   created_at, updated_at, reviewed_at, verified_at, retracted_at,
   text_hash
   ```
   Missing optional columns are replaced with `NULL` or `''` literals so the tuple shape stays consistent.

   Branch order matters:
   - If timestamp columns exist, `source_url` is selected directly. This assumes `_init_schema` has already added `source_url`.
   - If timestamps do not exist but `source_url` does, timestamps are replaced with `''`.
   - If neither exists, both are replaced.

4. Fetch all node rows with `cursor.fetchall()`.

5. Inspect `justifications` columns and detect whether `content_hash` exists.

6. Load justifications:
   ```sql
   SELECT node_id, type, antecedents_json, outlist_json, label,
          [content_hash or '']
   FROM justifications
   ORDER BY rowid
   ```
   Build `justifications_by_node: dict[str, list[Justification]]` by decoding `antecedents_json` and `outlist_json` with `json.loads`.

7. Build `Node` objects directly:
   - Unpack the 15-column row.
   - Use `justifications_by_node.get(nid, [])`.
   - Decode `metadata_json` if non-empty, otherwise `{}`.
   - Normalize `None`-ish fields with `or ""` for `source_url`, `text_hash`, and timestamps.
   - Assign directly:
     ```python
     network.nodes[nid] = node
     ```

8. Rebuild the dependent index:
   ```python
   network._rebuild_dependents()
   ```
   This is necessary because nodes were inserted directly, bypassing `Network.add_node`.

9. Load nogoods:
   ```sql
   SELECT id, nodes_json, discovered, resolution FROM nogoods
   ```
   Decode `nodes_json` and append `Nogood` objects to `network.nogoods`.

10. Load network metadata:
    ```sql
    SELECT key, value FROM network_meta
    ```
    For each row, skip `key == "next_nogood_id"`; copy all other keys into `network.meta`.
    The entire block is wrapped in a broad `try/except` that silently ignores failures.

11. Load repos:
    ```sql
    SELECT name, path FROM repos
    ```
    Insert into `network.repos`.
    Also wrapped in a broad `try/except` with silent failure.

12. Load propagation log:
    ```sql
    SELECT timestamp, action, target, value
    FROM propagation_log
    ORDER BY rowid
    ```
    Append dicts to `network.log`.

13. Return the `Network`.

## Side Effects

- Reads from SQLite.
- Mutates the newly created `Network`:
  - populates `network.nodes`
  - attaches `Justification` objects to nodes
  - calls `network._rebuild_dependents()`
  - populates `network.nogoods`
  - populates `network.meta`
  - populates `network.repos`
  - populates `network.log`
- Does **not** write to the database, commit, or close the connection.
- Does **not** invoke `Network.add_node`, so any validation/logging/side effects of `add_node` are skipped.
- Does **not** read `node_tags` or `node_sources`; those normalized tables are not merged into the returned `Network`.

## Error Handling

- Missing optional columns are partially handled by `PRAGMA table_info` and literal substitutions.
- Missing `network_meta` or `repos` tables (or any error while reading them) are silently swallowed via `except Exception: pass`.
- Missing `propagation_log` is not caught; it would raise `sqlite3.OperationalError`.
- Invalid JSON in `justifications`, `nodes.metadata_json`, or `nogoods.nodes_json` is not caught and raises `json.JSONDecodeError`.
- Orphan justifications do not raise an error; they are simply never attached to a node.
- The function does not validate referential integrity, metadata types, or that decoded JSON has the expected shape.

## Usage Patterns

Typical usage:

```python
storage = Storage("network.db")
network = storage.load()
```

Callers typically use this to hydrate a `Network` before querying, mutating, or saving again. Because `load` bypasses `add_node`, callers should not assume that any validation or logging performed by `add_node` has occurred. If the database may be untrusted or corrupt, callers should catch `sqlite3.Error` and `json.JSONDecodeError`.

After loading, the caller can mutate the network and call `storage.save(network)`. Note that external tags/sources stored in `node_tags` / `node_sources` are preserved by `save`, but they are not loaded into `network.metadata` by `load`.

## Dependencies

- `sqlite3` for connection, `PRAGMA`, and queries.
- `json` for serialized fields.
- `Node`, `Justification`, and `Nogood` model constructors.
- `Network` constructor and private method `Network._rebuild_dependents`.
- SQLite schema conventions defined by `SCHEMA` and migrations in `_init_schema`.
- No external libraries beyond the Python standard library.

## Topics to Explore

- [file] `reasons/network.py` — Understand `Network`, especially `_rebuild_dependents` and how `next_nogood_id` is derived.
- [function] `reasons/storage.py:save` — The inverse persistence path and how it synchronizes nodes, justifications, tags, and sources.
- [file] `tests/test_storage.py` — Roundtrip and migration tests that exercise `load` against real schemas.
- [general] `next-nogood-id-derivation` — Why `load` skips `next_nogood_id` and how the network avoids ID collisions afterwards.
- [general] `external-tag-and-source-persistence` — How `node_tags` / `node_sources` are preserved by `save` but not loaded into `Network`.

## Beliefs

- `load-bypasses-add-node` — `load` constructs `Node` objects and assigns them directly to `network.nodes`, then calls `network._rebuild_dependents()`; it never invokes `Network.add_node`.
- `load-swallows-meta-and-repos-errors` — Exceptions from reading `network_meta` and `repos` are caught and ignored, so `load` can succeed even if those tables are missing or unreadable.
- `load-propagates-json-errors` — Invalid JSON in `justifications`, `nodes.metadata_json`, or `nogoods.nodes_json` raises `json.JSONDecodeError` because those `json.loads` calls are not guarded.
- `load-skips-next-nogood-id` — Any `network_meta` row with key `next_nogood_id` is explicitly skipped and is not added to `network.meta`.
- `load-drops-orphan-justifications` — Justifications whose `node_id` does not match a loaded node are placed in the temporary `justifications_by_node` map but never attached to a node and do not raise an error.
