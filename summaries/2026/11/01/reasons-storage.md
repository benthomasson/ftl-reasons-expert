---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/storage.py

# `reasons/storage.py` — SQLite Persistence for the Dependency Network

## Purpose

This file is the **persistence layer** for the `reasons` knowledge/belief network. It owns the mapping between the in-memory `Network` object (nodes, justifications, nogoods, propagation log, metadata, repos) and a single SQLite database file. Everything else in the codebase — the API server, the CLI, the MCP server, the import/export pipelines — reads and writes the network exclusively through `Storage`.

Its single responsibility: **serialize a `Network` to disk and reconstruct it faithfully**, with schema migration for databases created by older versions.

## Key Components

### `SCHEMA` (module-level constant)

The DDL for the core tables:

| Table | Role |
|-------|------|
| `nodes` | One row per belief node (id, text, truth_value, source metadata, timestamps, metadata_json) |
| `justifications` | Inference rules (antecedents → consequent) attached to a node |
| `nogoods` | Conflict sets discovered during propagation |
| `repos` | Named repository paths |
| `propagation_log` | Append-only audit trail of truth-value changes |
| `network_meta` | Key-value store for schema version, project name, counters |
| `nodes_fts` | FTS5 virtual table for full-text search over node text |
| `node_tags` | Many-to-many node ↔ tag (with `access:` prefix for access-control tags) |
| `node_sources` | Structured source citations per node (type, ref, URL, hash, pinned SHA/lines) |

### `PROPOSALS_SCHEMA` (module-level constant)

DDL for a `proposals` table (retract/supersede/add/nogood actions with status lifecycle). **Not created by `Storage._init_schema`** — it's defined here for convenience but is likely applied elsewhere (e.g., `propose_update.py` or the API layer). Worth knowing it's *available* but *not owned* by this class.

### `Storage` class

| Method | Contract |
|--------|----------|
| `__init__(db_path, project_name="")` | Opens (or creates) the SQLite file, sets WAL + foreign keys, runs schema + migrations. Sets `self._is_new` based on file existence. |
| `_init_schema()` | Executes `SCHEMA`, then runs column-level migrations (ALTER TABLE ADD COLUMN) and data migrations (populate `node_tags`/`node_sources` from legacy `metadata_json`). Seeds `network_meta` for new databases. |
| `save(network: Network)` | **Full-snapshot write**: within one `with self.conn:` transaction, deletes all rows from every table, re-creates the FTS table, re-inserts everything from the in-memory `network`. Preserves externally-added tags/sources that bypassed the `Network` object. |
| `load() -> Network` | Reconstructs a `Network` by querying each table, assembling `Node`/`Justification`/`Nogood` objects, and calling `network._rebuild_dependents()`. Returns a fresh object; the caller owns it. |
| `close()` | Closes the underlying `sqlite3.Connection`. |

## Patterns

1. **Full-snapshot save (delete-all + reinsert).** No incremental diffing. This is intentional and safe for "small networks" (the comment says so). The transaction boundary (`with self.conn:`) guarantees atomicity — either the whole network is persisted or none of it is.

2. **WAL journal mode.** Set in `__init__`. Allows a reader (e.g., the API serving a `load()`) to proceed concurrently with a writer (`save()`). Critical for the multi-process `api.py` / `mcp_server.py` setup.

3. **Inline migration strategy.** `_init_schema()` uses `PRAGMA table_info` to check for missing columns and issues `ALTER TABLE ADD COLUMN` as needed. For data migrations (tags, sources), it checks whether the *table* existed before `SCHEMA` ran (via `pre_tables` set) and back-fills from `metadata_json`. No Alembic, no versioned migration files — pragmatic for a single-developer tool.

4. **External-write preservation.** `save()` explicitly snapshots `node_tags` and `node_sources` *before* clearing, then re-inserts them *after* the network-derived rows. The comment explains why: `add_tags()` / `add_source()` in other modules write directly to these tables without going through the in-memory `Network`. This is the most subtle part of `save()` — a regression here would silently lose tags/sources added via the API.

5. **Backwards-compatible `load()`.** The column-detection cascade (checking `has_source_url`, `has_timestamps`, `has_supporting`, `has_text_hash`) builds a dynamic SELECT that works against databases from any schema era. The `try/except` blocks around `network_meta`, `repos`, and `propagation_log` tolerate tables that don't exist in very old databases.

6. **Deterministic justification ordering.** `load()` selects justifications with `ORDER BY rowid`, preserving the insertion order that `save()` used. This matters because `Node.justifications` is a list, and `supporting_justification` is an integer index into it.

## Dependencies

### Imports (this file depends on)

- **`sqlite3`** — stdlib; sole persistence engine.
- **`Node`, `Justification`, `Nogood`** — data classes from the package root (`reasons/__init__.py`). These define the shape of what gets serialized.
- **`SCHEMA_VERSION`, `build_meta`** from `.metadata` — versioning constant stamped into `network_meta`. (`build_meta` is imported but only used implicitly via `SCHEMA_VERSION`; `build_meta` may be a vestigial import or used in a code path I don't see.)
- **`Network`** from `.network` — the aggregate object that `save()` reads and `load()` constructs.

### Dependents (imported by)

- **`reasons/api.py`** — the HTTP API creates a `Storage` instance per request/project to load and save networks.
- **`reasons_lib/storage.py`** — a mirror copy for the "library" distribution (the repo has both `reasons/` and `reasons_lib/` trees).
- **~15 test files** — every test that touches persistence goes through `Storage`.

## Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  save(network)                                                  │
│                                                                 │
│  BEGIN TRANSACTION                                               │
│  ├─ Snapshot ext_tags, ext_sources (preserve external writes)   │
│  ├─ DELETE FROM node_tags / node_sources / justifications       │
│  ├─ DROP + CREATE nodes_fts                                     │
│  ├─ DELETE FROM nodes / nogoods / repos / propagation_log / meta│
│  ├─ INSERT nodes (from network.nodes.values())                  │
│  │   └─ INSERT nodes_fts row per node                           │
│  │   └─ INSERT justifications per node                          │
│  ├─ INSERT node_tags (access: + custom)                         │
│  ├─ INSERT node_sources (from node.source)                      │
│  ├─ RESTORE ext_tags for surviving node_ids                     │
│  ├─ RESTORE ext_sources for surviving node_ids (dedup)          │
│  ├─ INSERT nogoods                                              │
│  ├─ INSERT network_meta (merge network.meta + defaults)         │
│  ├─ INSERT repos                                                │
│  └─ INSERT propagation_log                                      │
│  COMMIT                                                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  load() → Network                                               │
│                                                                 │
│  1. PRAGMA table_info to detect available columns               │
│  2. SELECT nodes (dynamic column list) → node_rows              │
│  3. SELECT justifications ORDER BY rowid → dict[node_id][list]  │
│  4. Construct Node objects directly (NOT via network.add_node)  │
│  5. network._rebuild_dependents()  ← rebuilds reverse index     │
│  6. SELECT nogoods → append to network.nogoods                  │
│  7. SELECT network_meta → populate network.meta                 │
│  8. SELECT repos → populate network.repos                       │
│  9. SELECT propagation_log ORDER BY rowid → network.log         │
│  10. return network                                              │
└─────────────────────────────────────────────────────────────────┘
```

## Invariants

- **Atomicity of `save()`:** All writes happen inside a single `with self.conn:` block. SQLite's implicit transaction means a crash mid-save leaves the database in the previous consistent state.
- **Foreign key integrity:** `PRAGMA foreign_keys=ON` ensures `justifications.node_id` and `node_tags.node_id` / `node_sources.node_id` always reference existing nodes. (Note: SQLite only enforces this at the DML level; `DELETE FROM nodes` without `CASCADE` would violate FK — but `save()` deletes children before parents.)
- **Justification ordering is preserved:** `save()` inserts justifications in list order; `load()` reads `ORDER BY rowid`. The integer `supporting_justification` index on a node must align with this ordering.
- **FTS index is always rebuilt on save:** Dropping and recreating `nodes_fts` avoids stale entries from deleted/modified nodes.
- **`_rebuild_dependents()` is called after load:** The dependent index (node → nodes that list it as antecedent) is not stored in the DB; it's always recomputed. This is the canonical method and must stay in sync with the justification data.

## Error Handling

- **`save()`** has no explicit try/except. Any `sqlite3.IntegrityError`, `OperationalError`, or disk-full condition propagates to the caller. The `with self.conn:` context manager rolls back the transaction on exception, so a failed save leaves the prior state intact.
- **`load()`** wraps the `network_meta`, `repos`, and (implicitly) the FTS-related queries in bare `except Exception: pass` to tolerate missing tables in databases created before those tables were added. This is a deliberate backwards-compatibility shim — it means `load()` will silently return a `Network` with empty `meta`/`repos` for very old databases rather than crashing.
- **`_init_schema()` migrations** are silent — no logging, no version check beyond column presence. If a migration fails (e.g., disk full during ALTER TABLE), the exception propagates from `executescript` or the individual `execute` calls.
- **No retry logic, no connection pooling.** Each `Storage` instance holds one connection. The caller (API server, CLI) is responsible for lifecycle management (`close()`).

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` aggregate that `Storage` serializes; understand `add_node`, `_rebuild_dependents`, and how `nodes`, `nogoods`, `meta`, `repos`, `log` interact before modifying the save/load logic.
- [file] `reasons/api.py` — The primary consumer of `Storage`; shows how WAL mode enables concurrent reads during a `save()`, and how the external-write pattern (tags/sources written directly to the DB) is exercised.
- [function] `reasons/storage.py:save` — The external-tag/source preservation logic (lines ~170–220) is the most bug-prone section; `tests/test_storage.py` and `tests/test_pin_sources.py` cover the edge cases.
- [file] `reasons/metadata.py` — Defines `SCHEMA_VERSION` and `build_meta`; understanding the versioning scheme is prerequisite to adding new columns or changing the serialization format.
- [general] `proposals-lifecycle` — `PROPOSALS_SCHEMA` is defined here but never created by `Storage`; trace where it's actually applied (likely `propose_update.py` or `api.py`) to understand the proposal→accept/reject flow.

---

## Beliefs

- `save-is-atomic` — A call to `Storage.save()` either fully persists the network or leaves the database in its prior state; partial writes are impossible because all DML is wrapped in a single `with self.conn:` transaction.
- `load-rebuilds-dependents` — `Storage.load()` never stores or restores the dependent index; it calls `network._rebuild_dependents()` after node reconstruction, making the justification tables the single source of truth for antecedent→consequent links.
- `external-writes-bypass-network` — Tags and sources can be written directly to `node_tags` / `node_sources` (e.g., via `add_tags()` in the API) without updating the in-memory `Network`; `save()` must snapshot and restore these rows, and a missing restore silently loses data.
- `fts-rebuilt-on-every-save` — The `nodes_fts` FTS5 table is dropped and recreated on every `save()` call, guaranteeing it never contains stale entries from removed or modified nodes.
- `schema-migration-is-column-level` — `_init_schema()` migrates by checking `PRAGMA table_info` and issuing `ALTER TABLE ADD COLUMN`; there is no central version gate, so a database can have any subset of the "new" columns and `load()` must handle each combination.
