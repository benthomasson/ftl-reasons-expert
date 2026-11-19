---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/storage.py

# `reasons/storage.py`

## Purpose

This file owns **all persistence for a `Network`**. It is the single source of truth for how the in-memory dependency network (nodes, justifications, nogoods, propagation history, tags, sources, repos, metadata) maps to a SQLite database.

It is deliberately the *only* module that talks to SQLite for the core graph — a classic repository pattern sitting between the domain object `Network` and a `.db` file. Notably, it does **not** own the `proposals` table: `PROPOSALS_SCHEMA` is defined here but never applied by `Storage`, so proposal persistence is handled elsewhere (see `reasons/propose_update.py`).

---

## Key Components

### Constants

- **`SCHEMA`** — `CREATE TABLE IF NOT EXISTS` DDL for the whole core graph:
  - `nodes` — one row per belief, including `truth_value`, `supporting_justification`, source snapshot fields, `text_hash`, `metadata_json`, and timeline columns (`created_at`/`updated_at`/`reviewed_at`/`verified_at`/`retracted_at`).
  - `justifications` — `node_id` FK, `type`, `antecedents_json`, `outlist_json`, `label`, `content_hash`. Uses `rowid` as PK (no natural key).
  - `nogoods` — `id`, `nodes_json`, `discovered`, `resolution`.
  - `repos`, `propagation_log`, `network_meta` — supporting tables.
  - `nodes_fts` — FTS5 virtual table over `(id, text)` with a porter/unicode61 tokenizer and `-`/`_` as token chars (so identifiers split predictably).
  - `node_tags` and `node_sources` — normalized off the `nodes` row, indexed for lookups.
- **`PROPOSALS_SCHEMA`** — DDL for the `proposals` table. Declared here as a shared constant, but `_init_schema` never executes it. Any consumer must `executescript` it themselves.

### `Storage` class

**`__init__(db_path, project_name="")`**
- Stores path, computes `_is_new = not path.exists()` (so "newness" is determined by the file, not by whether tables are empty).
- Opens a `sqlite3` connection, enables **WAL** (concurrent readers) and **`PRAGMA foreign_keys=ON`** (FKs actually enforced).
- Calls `_init_schema()`.

**`_init_schema()`** — the workhorse for bootstrap **and** upgrade:
1. Snapshots the existing table names (`pre_tables`) *before* running `SCHEMA`, so it can detect which tables the DDL just created.
2. `executescript(SCHEMA)` — creates anything missing.
3. **Column-level migrations** via `PRAGMA table_info`: adds `source_url`, timestamps, `supporting_justification`, `text_hash` to `nodes`; adds `content_hash` to `justifications` with `ALTER TABLE ... ADD COLUMN`. This is additive-only (no destructive migrations).
4. **Data backfills** when `node_tags`/`node_sources` were just introduced: copies `metadata.access_tags` into `node_tags` (namespaced as `access:<tag>`) and copies each node's `source` into `node_sources`.
5. If `_is_new`, seeds `network_meta` with `schema_version`, `project_name` (falling back to the file stem), `created_at`, `updated_at`.
6. Commits.

**`save(network)`** — atomic full-rewrite:
- Runs inside `with self.conn:` so the entire rewrite is one transaction (ACID guarantee for the propagation cascade).
- **Snapshots** `node_tags` and `node_sources` into `ext_tags`/`ext_sources` *before* clearing. This is the crucial subtlety: helper methods like `add_tags()`/`add_source()` write directly to these tables, bypassing the in-memory `Network`, so the snapshot is how those writes survive a save.
- Clears everything: `node_tags`, `node_sources`, `justifications`, `nodes_fts` (drop + recreate), `nodes`, `nogoods`, `repos`, `propagation_log`, `network_meta`.
- Re-inserts every node row, its FTS row, and its justifications.
- Resyncs tags/sources from `node.metadata`: `access_tags` become `access:<t>`; plain `tags` are inserted verbatim; `node.source` produces a `node_sources` row and is recorded in `written_sources`.
- **Restores** the external tags for still-present nodes, and re-inserts external source rows unless a `(node.id, source)` pair was already written from the in-memory node (dedup via `written_sources`).
- Re-inserts nogoods, then rebuilds `network_meta` from `network.meta` with `setdefault` for `schema_version`, `project_name`, `created_at`, and forces `updated_at = now`.
- Re-inserts `repos` and `propagation_log`.

**`load() -> Network`** — tolerant read:
- Introspects `nodes` columns and builds one of four `SELECT` variants depending on whether `source_url`, timestamp columns, `supporting_justification`, and `text_hash` exist. This lets an old DB file be opened by current code without failing.
- Loads justifications separately and buckets them by `node_id`; constructed objects preserve `content_hash`.
- Builds `Node` objects directly and assigns into `network.nodes[nid]` — **deliberately bypassing `add_node`** so exact persisted state (truth value, timestamps, source) isn't re-derived. Then calls `network._rebuild_dependents()` to reconstruct the in-memory reverse index.
- Appends `Nogood` rows to `network.nogoods`.
- Loads `network_meta`, but **skips `next_nogood_id`** — that counter is left to be derived on the in-memory side from the loaded nogoods. This prevents ID collisions when the persisted (stringly-typed) counter disagrees with actual rows.
- Loads `repos` and `propagation_log`; both wrapped in `try/except Exception: pass` for old DBs missing those tables.

**`close()`** — thin `self.conn.close()`.

### Flow

```
Storage(path)  ──►  connect ──► PRAGMA WAL / FK ──► _init_schema()
                                                      ├── executescript(SCHEMA)
                                                      ├── ALTER TABLE migrations
                                                      ├── backfill node_tags / node_sources
                                                      └── seed network_meta (new DB only)

save(network)  ──► snapshot ext → DELETE all → recreate FTS → insert nodes/fts/justifications
                → sync tags/sources from metadata → restore ext → nogoods → meta → repos → log → COMMIT

load()         ──► introspect columns → SELECT nodes → bucket justifications
                → construct Nodes (direct) → _rebuild_dependents → nogoods → meta → repos → log
```

---

## Patterns & Conventions

- **Repository / DAO**. `Storage` is the persistence boundary; `Network` stays pure-Python.
- **Additive, self-healing migrations.** Upgrade logic checks `PRAGMA table_info` / `sqlite_master` and only ever `ALTER TABLE ... ADD COLUMN`, never drop or rename. No version-bump gate — it runs every open.
- **Full snapshot rewrite on save.** No diffing, no per-node `INSERT OR REPLACE`. Simplest to reason about for "small networks" (explicitly noted in a comment) at O(all-rows) cost per save.
- **Atomic batch via `with self.conn:`.** All the `DELETE` + `INSERT` pairs happen in one SQLite transaction.
- **Denormalization + normalization together.** `nodes.metadata_json` still contains `access_tags`, `tags`, `source_type`, `pinned_sha`, `pinned_lines`, *and* these are mirrored into `node_tags`/`node_sources`. `save` must therefore reconcile both representations.
- **Column-introspection for backward compatibility.** `load` selects different column lists per schema era rather than requiring a migration to read.
- **Namespacing convention.** `access_tags` are prefixed `access:` when written to `node_tags`; plain tags are not.

---

## Dependencies

**Depends on:**
- `json`, `sqlite3`, `datetime`, `pathlib` — standard library.
- `reasons/__init__.py` exports `Node`, `Justification`, `Nogood` — the domain model built here.
- `reasons/metadata.py`: `SCHEMA_VERSION`, `build_meta`. **Note: `build_meta` is imported but not used** in this file — likely residual from an earlier refactor.
- `reasons/network.py`: `Network`, and implicitly its `_rebuild_dependents()` method (private, called from `load`).

**Depended on by:**
- `reasons/api.py` — the public factory likely constructs `Storage` to load/save.
- `reasons_lib/storage.py` — identical copy in the library package (the two trees appear to mirror each other).
- Many tests: `test_storage.py`, `test_roundtrip.py`, `test_nogood_id.py`, `test_merkle.py`, `test_metadata_extended.py`, `test_import_agent.py`, `test_dialectical.py`, `test_outlist.py`, `test_source_type.py`, `test_api.py`, `test_cli.py`, `test_dependents_integrity.py`, `test_summarize.py`. Round-trip coverage is the main contract these tests encode.

---

## Invariants

1. **`save` is a complete replacement of the persisted graph.** Anything not present in `network.nodes` / `network.nogoods` / `network.repos` / `network.log` / `network.meta` after save is gone from disk.
2. **Tags and sources added out-of-band are preserved.** `add_tags()`/`add_source()` must keep writing to `node_tags`/`node_sources` directly; `save` relies on this and re-restores those rows for surviving node ids.
3. **Access tags are stored namespaced.** `metadata["access_tags"] = ["x"]` → `node_tags` row `("id", "access:x")`. Plain `metadata["tags"]` are unnamespaced.
4. **`node_sources` dedup key is `(node_id, source_ref)`.** External rows are skipped if the in-memory node already produced a row with that `source` string.
5. **FKs are enforced (`PRAGMA foreign_keys=ON`).** Insertion order matters: nodes before their tags/sources/justifications; deletion order must clear children first.
6. **`next_nogood_id` is never loaded from `network_meta`.** It is recomputed from live nogoods to avoid collisions.
7. **FTS is content-preserving only within a save cycle.** It is dropped and rebuilt every `save`; no incremental FTS maintenance.
8. **Loading constructs nodes directly.** No `add_node`, no re-validation of derived fields — the persisted snapshot is authoritative (which is why `_rebuild_dependents()` is required to fix the in-memory index).

---

## Error Handling

- **Migration/additive schema errors are avoided** by existence checks (`PRAGMA table_info`, `pre_tables`), not by catching exceptions.
- **Old-schema tables are swallowed**: `network_meta`, `repos`, and (implicitly) `propagation_log` loads are wrapped in broad `try/except Exception: pass`. A missing table on an old DB results in an empty section rather than a crash.
- **JSON decoding is not guarded.** `json.loads` on `metadata_json`, `antecedents_json`, `outlist_json`, `nodes_json` will raise `JSONDecodeError` if the DB has corrupt/legacy non-JSON payloads. There is no defensive parse helper (e.g. `json.loads(s or "{}")` is applied at the caller in some spots but not centralized).
- **Save errors propagate.** Any exception inside `with self.conn:` triggers SQLite rollback; there is no internal try/except that would mask it.
- **No explicit validation of `Node` fields on save.** `text NOT NULL` and `id PRIMARY KEY` are the only DB-level guards.

---

## Topics to Explore

- [file] `reasons/network.py` — Defines `Network`, its `_rebuild_dependents()` and nogood-counter logic that `load`/`save` depend on.
- [file] `reasons/propose_update.py` — Likely the consumer of `PROPOSALS_SCHEMA`; confirms where the proposals table actually gets created.
- [file] `reasons/metadata.py` — Source of `SCHEMA_VERSION` and `build_meta` (imported but unused here); clarifies the intended metadata contract.
- [file] `tests/test_storage.py` — Encodes the round-trip and migration invariants this module must uphold.
- [general] `external-tag-source-preservation` — Trace `add_tags` / `add_source` callers to understand the out-of-band write pattern `save` is protecting against.

## Beliefs

- `save-replaces-whole-network` — `save()` deletes every row in every core table and re-inserts from the in-memory `Network` inside a single transaction.
- `save-preserves-external-tags-and-sources` — `save()` snapshots `node_tags`/`node_sources` before clearing and restores them for nodes still present.
- `access-tags-are-prefixed` — Access tags are inserted into `node_tags` as `access:<tag>`, while plain `metadata["tags"]` are inserted verbatim.
- `load-bypasses-add-node` — `load()` constructs `Node` objects directly and calls `network._rebuild_dependents()` instead of using `Network.add_node`.
- `next-nogood-id-skipped-on-load` — `load()` explicitly skips the `next_nogood_id` key in `network_meta`, leaving the counter to be recomputed from loaded nogoods.
- `proposals-schema-unused-here` — `PROPOSALS_SCHEMA` is declared in this file but never executed by `Storage._init_schema`.
- `build-meta-import-unused` — `build_meta` is imported from `reasons.metadata` but never referenced in this file.
