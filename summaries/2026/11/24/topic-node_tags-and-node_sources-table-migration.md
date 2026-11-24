---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How tags/sources moved out of the serialized `Network` into side tables, and which write paths must stay in sync.

# How tags/sources moved out of `Network` into side tables

## The two representations

There are now **two places** the same data lives, and the explanation is really about keeping them aligned.

**In-memory `Network`** (`reasons/network.py:17`) — the class itself has only `nodes`, `nogoods`, `repos`, `log`, and `meta`. It has **no** `tags` dict and **no** `sources` dict. Tags and sources are attributes on each `Node`:

- Tags: `node.metadata["access_tags"]` (a plain list of strings), plus arbitrary `node.metadata["tags"]`
- Sources: `node.source`, `node.source_url`, `node.source_hash`, plus `node.metadata["source_type"]`, `node.metadata["pinned_sha"]`, `node.metadata["pinned_lines"]`

That's what gets mirrored into the `nodes.metadata_json` JSON column on write, so it *looks* like it "serializes with the network."

**SQLite side tables** — `reasons/storage.py:SCHEMA` defines dedicated, indexed tables:

```
node_tags (node_id, tag)                         -- PK (node_id, tag)
  CREATE INDEX idx_node_tags_tag                 -- lookup by tag
node_sources (node_id, source_type, source_ref,
              source_url, source_hash,
              pinned_sha, pinned_lines, label, added_at)
  CREATE INDEX idx_node_sources_node
```

The point of the split: `metadata_json` is a JSON blob — it can't be indexed or searched by tag. `node_tags` and `node_sources` are plain relational tables that can. The `node_tags.tag` column stores tags **with a namespace prefix** — `access:finance` for `access_tags`, and bare tag strings for `metadata["tags"]` (see the sync block in `Storage.save`).

## How the migration happened

There is no separate migration module — the migration is baked into `Storage._init_schema` in `reasons/storage.py`:

1. **Detect old databases first.** Before running `SCHEMA`, it captures `pre_tables = SELECT name FROM sqlite_master` and then executes `SCHEMA` (which contains `CREATE TABLE IF NOT EXISTS node_tags` / `node_sources`).
2. **One-shot backfill on upgrade.** If `node_tags` *wasn't* in `pre_tables`, it iterates every `nodes` row, reads `metadata_json`, and inserts `access:<tag>` into `node_tags`. If `node_sources` wasn't there, it iterates `nodes` and, for any row with a non-empty `source`, inserts a `node_sources` row using `source`, `source_url`, `source_hash` as `source_ref`/`source_url`/`source_hash`, plus `metadata["source_type"]`, `metadata["pinned_sha"]`, `metadata["pinned_lines"]`.
3. **On every subsequent `save()`, model-to-table sync is rewritten from scratch** (see below).

So the "move" was really an additive mirror: the source of truth in the persisted schema is still `nodes.metadata_json` + the `source*` columns, and `node_tags`/`node_sources` are a derived, queryable projection that `load()` does **not** read from. If you deleted both side tables right now, `load()` would still return a fully consistent `Network`.

## The `save()` bridge (the part that has to stay in sync)

`Storage.save` (in `reasons/storage.py`) is the only place the two representations are reconciled in one transaction. It has three-phase behavior:

1. **Snapshot the side tables *before* clearing them.**
   ```
   ext_tags    = SELECT node_id, tag FROM node_tags
   ext_sources = SELECT ... FROM node_sources
   ```
   The comment is explicit: *"add_tags()/add_source() write directly to these tables, bypassing the in-memory Network, so we must preserve them."* This is the critical invariant — there are writers to `node_tags`/`node_sources` that **do not touch `Network`**, so `save()` treats that content as external state.
2. **Clear and rewrite** every table (`DELETE FROM node_tags`, `DELETE FROM node_sources`, `DELETE FROM nodes`, etc.) and re-insert from `network.nodes.values()`. For each node:
   - `node.metadata["access_tags"]` → `node_tags` rows prefixed `access:`
   - `node.metadata["tags"]` → `node_tags` rows as-is
   - `node.source` (non-empty) → one `node_sources` row, pulling `source_type`, `pinned_sha`, `pinned_lines` from metadata.
3. **Restore the snapshotted external rows** for node IDs that still exist, skipping ones that were already written from the in-memory model (`written_sources` set keyed on `(node_id, source)`, plus `INSERT OR IGNORE` for tags).

This is why `save()` is more than a naïve `DELETE + INSERT`: without the snapshot/restore dance, every save would silently drop tags/sources that were added via paths that bypass `Network`.

## Write paths that must stay in sync

Anything that adds, changes, or removes a tag or source has to pick a side and respect the contract. In the observed code:

| Writer | What it mutates | Where the truth lives |
|---|---|---|
| `Network.add_node` (`reasons/network.py:add_node`) | `node.metadata`, `node.source*` | memory + `metadata_json` |
| `Network.add_justification` → `_inherit_access_tags`, `_propagate_access_tags` | `node.metadata["access_tags"]` on self + dependents | memory only (re-projected on save) |
| `pin_sources`, `pin_update`, `pin_lines` (`reasons/check_stale.py`, tests in `tests/test_pin_sources.py`) | `node.metadata["pinned_sha"]`, `node.metadata["pinned_lines"]`, `node.source_hash`, `node.metadata["verified_at"]` | memory (re-projected into `node_sources.pinned_sha`/`pinned_lines` on save) |
| `Storage.save` | rewrites `node_tags` / `node_sources` from memory, then re-applies external rows | side tables |
| External `add_tags()` / `add_source()` (referenced only by comment in `save()`) | writes `node_tags`/`node_sources` **directly** | side tables only, until next save survives the snapshot |

If you add a new write path, the rules are:

- **Mutating tags/sources in memory must go through `Network`** (`add_node` metadata or `_inherit_access_tags`), because `add_justification` and `_propagate_access_tags` recompute the inherited set and will clobber a hand-poked list on the next call.
- **Mutating side tables directly is allowed but is by definition "external"** — it will only survive a `save()` if the writer is either (a) still in `ext_*` at snapshot time, or (b) since been mirrored back through a memory mutation.
- **`load()` does not read either side table**, so you can't rely on them for correctness — only for query performance (`idx_node_tags_tag`, `idx_node_sources_node`).

## What the observations are insufficient for

A few things are referenced but not shown:

- The actual `add_tags()` / `add_source()` functions that the `save()` comment describes — we only see the comment, not the implementation. Their exact conflict semantics (last-write-wins? merge?) need to be read directly.
- `reasons/pg.py` is 3,736 lines and only the first 700 are shown. The visible `PgApi` schema has **no** `rms_node_tags` / `rms_node_sources` tables, and `_add_node_raw` / `add_node` write `source`, `source_url`, `source_hash` into `rms_nodes` columns and `access_tags` into `metadata` JSONB. If a parallel PG side-table migration exists, it's past line 700.
- The `find_tag_source_side_tables` search returned **0 matches** despite `node_tags` / `node_sources` appearing throughout `storage.py` — that result contradicts the visible code and should be treated as a failed observation, not evidence the tables don't exist.
- Exact line numbers inside `storage.py` for `_init_schema` and `save` aren't pinpointed by the observations; the schema is near the top of the file and `_init_schema` / `save` follow shortly after.

## Topics to Explore

- [file] `reasons/storage.py` — Read `_init_schema`, `save`, and `load` end-to-end; the migration and snapshot/restore logic is easy to break.
- [function] `reasons/storage.py:Storage.save` — The snapshot → clear → rewrite → restore sequence is the sole reconciliation point between `Network` and the side tables.
- [function] `reasons/pg.py:PgApi._add_node_raw` — See how the PostgreSQL backend currently stores sources in `rms_nodes` columns instead of a side table; check whether a `rms_node_sources` equivalent exists later in the file.
- [general] `add_tags-add_source-writers` — Locate the `add_tags()` and `add_source()` functions named in the `save()` comment; their existence is why the external snapshot exists at all.
- [file] `reasons/check_stale.py` — `pin_sources`, `pin_update`, and `pin_lines` are the in-tree writers that mutate `metadata["pinned_sha"]`/`metadata["pinned_lines"]` and `source_hash`; they must run before `save()` to be picked up.

## Beliefs

- `network-serialization-excludes-tags-and-sources-as-collections` — `Network.__init__` defines `nodes`, `nogoods`, `repos`, `log`, `meta` only; tags and sources live per-node in `Node.metadata` / `Node.source*`.
- `storage-scHEMA-defines-node-tags-and-node-sources-tables` — `reasons/storage.py:SCHEMA` creates `node_tags` and `node_sources` as the queryable side tables, with `idx_node_tags_tag` and `idx_node_sources_node`.
- `storage-init-schema-backfills-side-tables-on-upgrade` — `Storage._init_schema` detects `node_tags`/`node_sources` missing before `SCHEMA` runs and backfills from `metadata_json` + `nodes.source*`.
- `save-preserves-external-side-table-writes` — `Storage.save` snapshots `node_tags`/`node_sources` before clearing, then restores rows whose `(node_id, source_ref)` was not re-derived from the in-memory network.
- `load-ignores-side-tables` — `Storage.load` reconstructs `Network` from `nodes` + `metadata_json` + `justifications`, never from `node_tags` or `node_sources`.
