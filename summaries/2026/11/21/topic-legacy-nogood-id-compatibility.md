---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Explore how old-format IDs are stored, displayed, and deduplicated alongside content-derived IDs.

## How old-format IDs coexist with content-derived IDs

The observed code has **two separate content-derived mechanisms** plus an import path for **old-format IDs**:

1. `content_hash` on justifications: a Merkle-derived value stored in SQLite and surfaced through the API.
2. Nogood IDs: generated from sorted node IDs, e.g. `nogood-a-b`, with numeric suffixes for duplicate node sets.
3. Old-format nogood IDs such as `nogood-005` / `nogood-010`, accepted by import paths and preserved alongside newly generated content-derived IDs.

One important distinction: the `legacy|old_id` search results are mostly about **supersession** (`old_id` is the node being superseded), not about an old ID storage format. Those matches should not be mistaken for legacy-ID migration logic.

### Storage of `content_hash` and old rows

`content_hash` is a first-class justification field:

- `reasons/__init__.py:23` defines `content_hash: str = ""`, so the in-memory model defaults to empty.
- `reasons/storage.py:43` creates the SQLite column as `content_hash TEXT DEFAULT ''`.
- `reasons/storage.py:153-154` adds the column if an existing database is missing it:
  ```python
  if "content_hash" not in j_cols:
      self.conn.execute("ALTER TABLE justifications ADD COLUMN content_hash TEXT DEFAULT ''")
  ```
- `reasons/storage.py:263-265` includes `content_hash` in justification inserts.
- `reasons/storage.py:390` checks whether loaded rows have the column, supporting older schemas.

So legacy rows can exist with `content_hash = ""`. That is the compatibility bridge for old-format data: old rows load, then missing hashes can be filled later.

### Computation and backfill of content-derived hashes

The Merkle module computes and verifies the content hash:

- `reasons/merkle.py:4` documents that a justification stores a `content_hash`, a Merkle root incorporating the node text and justification.
- `reasons/merkle.py:41` defines `_fresh_content_hash`.
- `reasons/merkle.py:59` defines `compute_content_hash`.
- `reasons/merkle.py:89-104` verifies the stored hash against a recomputed value and reports `stored_hash`.
- `reasons/merkle.py:125` and `:140-141` handle justifications whose hash is empty by computing it.

`Network` also backfills missing hashes:

- `reasons/network.py:130-134` imports `compute_text_hash, compute_content_hash` and fills empty `j.content_hash`.
- `reasons/network.py:513-518` does the same in another path.

So old justifications are not rejected for lacking `content_hash`; they are loaded, then hashed on demand or during network processing.

### Display of content hashes and deduplication metadata

Content hashes are included in API JSON output:

- `reasons/api.py:900` exposes `"content_hash": j.content_hash`.
- `reasons/api.py:1851` exposes the same in another serialization path.

Deduplication metadata is displayed through node metadata and exports:

- `api.mark_duplicate` at `reasons/api.py:1240` retracts the duplicate and stores the canonical relationship.
- `reasons/api.py:1267` sets `meta["duplicate_of"] = canonical_id`.
- `tests/test_deduplication.py` verifies that `show_node` and `export_network` expose `metadata["duplicate_of"]`, `metadata["superseded_by"]`, and `retract_reason`.
- `reasons/build_wiki.py:60-63` renders `duplicate_of` relationships into wiki output.

For nogood IDs specifically, the observations do **not** include a `show_nogood` or `list_nogoods` display function, so old-format nogood ID display behavior is not directly visible. What is visible is import and generation behavior.

### Old-format nogood IDs alongside content-derived IDs

The nogood ID tests are the clearest evidence:

- `tests/test_nogood_id.py` `test_id_from_sorted_nodes` shows content-derived ID generation:
  - `add_nogood(["b", "a"])` produces `"nogood-a-b"`.
- `test_duplicate_gets_suffix` shows duplicate node sets get numeric suffixes:
  - first `["a", "b"]` -> `"nogood-a-b"`
  - second `["a", "b"]` -> `"nogood-a-b-2"`
- `test_id_survives_save_load` shows generated IDs are persisted through `Storage.save` / `Storage.load`.

Old-format IDs are accepted during import:

- `TestImportJson.test_import_preserves_old_format_ids` imports a nogood with ID `"nogood-005"` for nodes `["a", "b"]`.
- `TestImportBeliefs.test_import_nogoods_preserves_old_format_ids` imports a belief format nogood ID `"nogood-010"` for nodes `"a, b"`.

After importing the old-format `"nogood-005"`, the JSON-import test then calls:

```python
result = api.add_nogood(["a", "b"], db_path=db)
assert result["nogood_id"] == "nogood-a-b"
```

So the observed contract is: old-format IDs are accepted at import time, while newly added nogoods use content-derived IDs. The test does not query `"nogood-005"` afterward, and the implementation of `import_json` / `add_nogood` is not shown, so the exact collision, replacement, or coexistence rules are not fully visible.

### Deduplication of beliefs versus nogoods

There are two different deduplication patterns in the observations:

**Belief/node deduplication** uses reviewed clusters and explicit metadata:

- `reasons/api.py:4976` defines `deduplicate`, described as finding clusters of IN beliefs with similar IDs or text.
- `reasons/api.py:5079` defines `write_dedup_plan` for human review.
- `reasons/api.py:5107` defines `parse_dedup_plan`.
- `reasons/api.py:5139` defines `apply_dedup_plan`, which rewrites justifications and retracts duplicates.
- `reasons/api.py:1240` `mark_duplicate` records `duplicate_of` and retracts the source node.
- The deduplication tests verify cascading retraction and metadata export/display.

This is the mechanism most directly relevant to deduplicating old IDs against content-derived or canonical IDs: the dedup plan clusters by similar ID or text, a human reviews it, and `apply_dedup_plan` rewrites/retracts. The observations do not show `content_hash` being used as the deduplication key.

**Nogood deduplication** uses content-derived IDs and suffixes:

- Same node set -> `"nogood-a-b"`.
- Repeated same node set -> `"nogood-a-b-2"`.

That is a deterministic content-ID scheme rather than the metadata-based `mark_duplicate` workflow.

### What is missing from the observations

The observations are insufficient to fully answer:

- How imported old IDs are stored in the `nogoods` table.
- Whether imported `"nogood-005"` remains queryable or is rewritten later.
- How `add_nogood` collides an old-format ID with a newly generated content-derived ID.
- Whether `deduplicate` uses `content_hash` or only IDs/text.
- How old-format node IDs, as opposed to nogood IDs, are handled.

## Topics to Explore

- [function] `reasons/api.py:add_nogood` — Determines the exact collision and suffix behavior when an old-format ID already exists for the same node set.
- [file] `reasons/import_beliefs.py` — Shows how old-format nogood IDs like `nogood-010` are parsed and stored.
- [function] `reasons/api.py:import_json` — Reveals whether imported old IDs are preserved verbatim, remapped, or both.
- [function] `reasons/merkle.py:compute_content_hash` — Explains the exact Merkle-root inputs and how verification behaves for legacy empty hashes.
- [function] `reasons/api.py:deduplicate` — Clarifies whether deduplication compares IDs, text, content hashes, or some combination.

## Beliefs

- `content-hash-column-defaults-empty` — `reasons/storage.py:43` defines `content_hash TEXT DEFAULT ''`, and `storage.py:153-154` migrates existing databases by adding that column.
- `missing-content-hash-is-backfilled` — `reasons/network.py:130-134` and `:513-518` compute `content_hash` when a justification has no stored hash.
- `nogood-ids-are-content-derived` — `tests/test_nogood_id.py` shows new nogoods derive IDs from sorted node IDs, producing `nogood-a-b`, with `-2` suffixes for duplicate node sets.
- `old-format-nogood-ids-are-import-preserved` — `tests/test_nogood_id.py` imports `nogood-005` and `nogood-010`, while a later `add_nogood(["a", "b"])` returns the content-derived `nogood-a-b`.
- `mark-duplicate-produces-duplicate-of-metadata` — `reasons/api.py:1267` writes `metadata["duplicate_of"]`, which is verified in `tests/test_deduplication.py` and rendered by `reasons/build_wiki.py:60-63`.
