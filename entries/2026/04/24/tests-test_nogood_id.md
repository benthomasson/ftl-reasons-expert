# File: tests/test_nogood_id.py

**Date:** 2026-04-24
**Time:** 17:01

# `tests/test_nogood_id.py` — Monotonic Nogood ID Generation

## Purpose

This file is a regression test suite for **issue #26**: a bug where nogood IDs could be reused after deletion. Nogoods are contradiction records — pairs (or sets) of beliefs that cannot all be true simultaneously. Each nogood gets a sequential ID like `nogood-001`, `nogood-002`, etc.

The original bug: IDs were derived from `len(self.nogoods) + 1`, so if you had two nogoods, deleted one, then added another, the new one would get the same ID as the deleted one. The fix introduced a persisted monotonic counter (`_next_nogood_id`) on `Network` that only ever increments. This file proves that fix works across every surface that creates or loads nogoods.

## Key Components

### `TestMonotonicCounter` — In-memory counter behavior

Three tests that verify the counter works correctly without persistence:

- **`test_ids_increment`**: Baseline — two nogoods get `nogood-001` and `nogood-002`, counter advances to 3.
- **`test_delete_last_does_not_reuse_id`**: The core regression test. Deletes the second nogood, adds a third — it must get `nogood-003`, not `nogood-002`.
- **`test_delete_all_does_not_reset`**: Clears the entire nogoods list. Counter stays at 2; next nogood gets `nogood-002`, not `nogood-001`.

### `TestPersistence` — Counter survives save/load cycles

Four tests covering SQLite round-tripping:

- **`test_counter_survives_save_load`**: Save a network with one nogood, reload it — counter is still 2.
- **`test_counter_persists_after_deletion`**: Delete a nogood *before* saving. After reload, counter is still 3 (reflects the high-water mark, not current count).
- **`test_old_db_without_meta_table`**: Simulates upgrading a database created before this fix by dropping the `network_meta` table. The loader must gracefully default to `_next_nogood_id = 1` when no nogoods exist.
- **`test_old_db_with_nogoods_derives_counter`**: Same migration scenario, but with existing nogoods. The loader must scan existing nogood IDs and derive the counter (max existing ID + 1 = 3), not naively default to 1.

### `TestImportJson` — JSON import updates counter

Imports a JSON file containing a nogood with ID `nogood-005`. After import, the next nogood added must get `nogood-006` — the counter must jump to accommodate imported IDs.

### `TestImportBeliefs` — Markdown import updates counter

Imports a beliefs markdown file containing `nogood-010`. After import, `_next_nogood_id` must be 11, and the next added nogood gets `nogood-011`.

## Patterns

- **Fixture isolation**: The `db` fixture creates a fresh SQLite database in `tmp_path` for each test. Persistence tests skip the fixture and manage `Storage` directly to control save/load boundaries.
- **Simulate schema migration**: Tests drop the `network_meta` table via raw SQL (`DROP TABLE IF EXISTS network_meta`) to simulate loading a database from before the fix existed. This is a common pattern for testing backwards compatibility without maintaining actual old database snapshots.
- **Two-phase storage**: Each persistence test opens a `Storage`, saves, closes it, then opens a *new* `Storage` on the same path. This proves the counter is actually written to disk, not just held in memory.

## Dependencies

**Imports:**
- `reasons_lib.network.Network` — the in-memory dependency graph that owns `_next_nogood_id` and `nogoods`
- `reasons_lib.storage.Storage` — SQLite persistence layer; must serialize/deserialize the counter via `network_meta` table
- `reasons_lib.api` — functional API layer (`init_db`, `add_node`, `add_nogood`, `import_json`)
- `reasons_lib.import_beliefs.import_into_network` — markdown import that must update the counter

**Imported by:** Nothing — this is a test file.

## Flow

Each test follows the same shape:

1. Create a `Network` (or initialize via `api.init_db`)
2. Add nodes (nogoods require their constituent nodes to exist)
3. Add nogoods, possibly delete some
4. Assert `_next_nogood_id` and the `.id` field of newly created nogoods
5. For persistence tests: save → close → reopen → load → assert counter survived

The import tests add a wrinkle: they inject nogoods with *arbitrary* IDs (e.g., `nogood-005`, `nogood-010`) and verify the counter jumps past the highest imported ID.

## Invariants

1. **Monotonicity**: `_next_nogood_id` never decreases, regardless of deletions or clears.
2. **Uniqueness**: No two nogoods ever share the same ID within a network's lifetime.
3. **Persistence**: The counter round-trips through SQLite via `network_meta`.
4. **Backwards compatibility**: Old databases without `network_meta` load successfully; the counter is derived from existing nogood IDs (max ID + 1) or defaults to 1 if none exist.
5. **Import coherence**: Both JSON and markdown import paths must update `_next_nogood_id` to be at least max(imported ID) + 1.

## Error Handling

There is no explicit error handling in this file — it's a test suite that relies on assertion failures. The implicit contract is that `Storage.load()` must not crash on databases missing the `network_meta` table (the backwards-compatibility tests verify this).

---

## Topics to Explore

- [function] `reasons_lib/network.py:add_nogood` — Where the counter is incremented and the `nogood-NNN` ID is formatted
- [file] `reasons_lib/storage.py` — How `network_meta` is created, written, and read during save/load, including the migration fallback
- [function] `reasons_lib/api.py:import_json` — How imported nogood IDs are parsed and the counter is synchronized
- [file] `reasons_lib/import_beliefs.py` — Markdown parser that extracts nogood IDs like `nogood-010` and must update the counter
- [general] `nogood-backtracking` — How nogoods trigger `find_culprits` and entrenchment-based premise retraction (see `test_backtracking.py`)

## Beliefs

- `nogood-id-monotonic` — `Network._next_nogood_id` only ever increases; deletions and clears do not decrement it
- `nogood-id-persisted-in-meta` — The nogood counter is stored in a `network_meta` SQLite table and survives save/load cycles
- `nogood-id-backwards-compat` — Databases lacking `network_meta` load without error; the counter is derived from max existing nogood ID or defaults to 1
- `nogood-id-import-sync` — Both `import_json` and `import_into_network` must advance `_next_nogood_id` past the highest imported nogood ID

