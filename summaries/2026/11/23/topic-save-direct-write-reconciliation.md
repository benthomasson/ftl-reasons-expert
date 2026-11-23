---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search for the actual `save` implementation and compare its tag/source handling to the direct writes performed by `add_tags` / `add_source`.

## Actual `save` implementation

The real `save` method is **not** in `reasons_lib/storage.py`; that file is only a compatibility re-export:

```python
from reasons.storage import *  # noqa: F401,F403
```

The actual implementation is:

- `reasons/storage.py:206` — `def save(self, network: Network) -> None:`

The earlier `find_save_definition` search returned zero definitions likely because it did not match an indented method definition. The grep for `def (save|add_tags|add_source)\b` did find it at `reasons/storage.py:206`.

## How `save` handles tags and sources

`Storage.save()` uses a full rewrite strategy:

1. Snapshot existing rows from `node_tags` and `node_sources`.
2. Delete rows from `node_tags`, `node_sources`, `justifications`, `nodes_fts`, `nodes`, `nogoods`, `repos`, `propagation_log`, and `network_meta`.
3. Reinsert nodes and FTS rows from the in-memory `Network`.
4. Continue reinserting justifications and other state.

The critical part is the comment and snapshot at the beginning of `save`:

```python
# Snapshot externally-added tags/sources before clearing.
# add_tags()/add_source() write directly to these tables,
# bypassing the in-memory Network, so we must preserve them.
ext_tags = self.conn.execute(
    "SELECT node_id, tag FROM node_tags"
).fetchall()
ext_sources = self.conn.execute(
    "SELECT node_id, source_type, source_ref, source_url, "
    "source_hash, pinned_sha, pinned_lines, label, added_at "
    "FROM node_sources"
).fetchall()
```

So `save` explicitly accounts for the fact that `node_tags` and `node_sources` may contain rows that are **not represented in the in-memory `Network` object**.

## Comparison to direct writes from `add_tags` / `add_source`

The API functions are:

- `reasons/api.py:938` — `def add_tags(node_id: str, tags: list[str], ...)`
- `reasons/api.py:1007` — `def add_source(node_id: str, source_ref: str, source_type: str = "", ...)`

Based on the `save` comment, these functions write directly to SQLite tables:

- `add_tags` writes directly to `node_tags`.
- `add_source` writes directly to `node_sources`.

They bypass the in-memory `Network`. That creates a two-source persistence problem:

- `Network` is the source of truth for node fields, justifications, etc.
- `node_tags` and `node_sources` may contain DB-only mutations made by `add_tags` / `add_source`.

A naive `save()` that deletes all tables and rewrites from `Network` would erase those direct writes. `save()` avoids that by snapshotting `node_tags` and `node_sources` before deleting them, with the stated intent of preserving them.

There is also a source-data duality:

- `save()` writes legacy source fields from the node object into `nodes.source`, `nodes.source_url`, and `nodes.source_hash`.
- Separately, `save()` snapshots normalized rows from `node_sources`, including `source_type`, `source_ref`, `source_url`, `source_hash`, `pinned_sha`, `pinned_lines`, `label`, and `added_at`.

So source information exists both in legacy node columns and in the normalized `node_sources` table.

## What is missing

The provided excerpt stops at `reasons/storage.py:260`, at:

```python
for j in node.justifications:
```

That means we can see the snapshot and deletion of `ext_tags` / `ext_sources`, but **not the reinsertion logic**. We also do not have the bodies of `add_tags` and `add_source`, only their signatures.

Therefore we can confidently say that `save()` intends to preserve direct tag/source writes, but we cannot confirm from the provided observations:

- whether `ext_tags` and `ext_sources` are reinserted verbatim,
- whether they are merged with tags/sources derived from `Network`,
- how duplicates are handled,
- whether `add_source` also updates legacy `nodes.source*` columns.

## Topics to Explore

- [file] `reasons/storage.py` — Read the rest of `save()` after line 260 to verify how `ext_tags` and `ext_sources` are restored.
- [function] `reasons/api.py:add_tags` — Inspect the direct SQL write path and transaction behavior for `node_tags`.
- [function] `reasons/api.py:add_source` — Inspect how `node_sources` rows are constructed and whether legacy node source fields are updated.
- [general] `tag-source-persistence-invariant` — Determine whether DB-only tags/sources must survive every `save()` and how they reconcile with `Network` state.
- [file] `reasons_lib/storage.py` — Confirm that this is only a compatibility shim and not the implementation location.

## Beliefs

- `save-is-defined-on-storage-class` — `save` is implemented as `Storage.save` at `reasons/storage.py:206`, not as a top-level function.
- `save-preserves-direct-tag-source-writes` — `save` snapshots `node_tags` and `node_sources` before clearing tables because `add_tags` and `add_source` write directly to the database.
- `add-tags-bypasses-network-state` — `add_tags` writes directly to `node_tags`, bypassing the in-memory `Network`.
- `add-source-bypasses-network-state` — `add_source` writes directly to `node_sources`, bypassing the in-memory `Network`.
- `source-data-has-dual-representation` — Source data is stored both in legacy `nodes.source*` columns and in normalized `node_sources` rows.
- `save-reinsertion-not-visible` — The provided excerpt does not show reinsertion of snapshotted `ext_tags` / `ext_sources`; that logic occurs after line 260.
