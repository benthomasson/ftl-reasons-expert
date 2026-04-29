# File: tests/test_nogood_id.py

**Date:** 2026-04-29
**Time:** 17:06

## Purpose

`tests/test_nogood_id.py` is a regression test suite for **issue #26** — a bug where nogood IDs could be reused after deletion. The root cause: IDs were derived from `len(self.nogoods) + 1`, so deleting a nogood and adding a new one would collide with the deleted ID. The fix introduced a monotonic counter (`_next_nogood_id`) on the `Network` object that only ever increases.

This file's job is to prove that the counter behaves correctly across five axes: basic incrementing, deletion resilience, persistence through save/load, backward compatibility with old databases, and correct counter derivation during JSON and belief imports.

## Key Components

### `TestMonotonicCounter` (in-memory correctness)

Three tests that exercise the counter without touching persistence:

- **`test_ids_increment`** — Baseline: two nogoods get sequential IDs `nogood-001`, `nogood-002`, counter advances to 3.
- **`test_delete_last_does_not_reuse_id`** — The core regression case. Deletes the second nogood, adds a third — asserts the new one gets `nogood-003`, not `nogood-002`.
- **`test_delete_all_does_not_reset`** — Clears the entire list, confirms the counter stays at 2 and the next nogood gets `nogood-002`.

### `TestPersistence` (save/load round-trip)

Four tests that serialize to SQLite via `Storage` and reload:

- **`test_counter_survives_save_load`** — Save a network with one nogood, reload, counter is still 2.
- **`test_counter_persists_after_deletion`** — Delete a nogood before saving, reload, counter still reflects the high-water mark (3), and the next add produces `nogood-003`.
- **`test_old_db_without_meta_table`** — Simulates a database created before the fix by dropping `network_meta`. On load, the counter falls back to 1 (no nogoods to derive from).
- **`test_old_db_with_nogoods_derives_counter`** — Same simulation but with existing nogoods. The loader must parse existing nogood IDs to derive the counter (3), not blindly default to 1. Also verifies that saving afterward works (the upgrade path).

### `TestImportJson`

- **`test_import_json_updates_counter`** — Imports a JSON file containing `nogood-005`. The next `add_nogood` call must produce `nogood-006`, meaning `import_json` scanned imported IDs and bumped the counter.

### `TestImportBeliefs`

- **`test_import_nogoods_updates_counter`** — Imports markdown-formatted nogoods containing `nogood-010`. The counter must advance to 11, and the next add must produce `nogood-011`.

### Fixtures

- **`db`** — Creates a fresh SQLite database in a `tmp_path`, calls `api.init_db`, returns the path string. Used by `TestImportJson` and `TestImportBeliefs`.

## Patterns

**Regression-driven testing.** Every test class targets a specific failure mode of the old `len()+1` approach. The structure is: set up the precondition that would trigger the bug, then assert the fix holds.

**Simulation of schema migration.** The `test_old_db_*` tests manually `DROP TABLE network_meta` to simulate databases created before the fix. This is a pragmatic alternative to maintaining actual old schema fixtures — it tests the migration path by creating the missing-table condition directly.

**Layered testing.** The suite moves from pure in-memory (`Network` only) to persistence (`Storage` round-trip) to import paths (`import_json`, `import_beliefs`). Each layer adds a new integration boundary where the counter could break.

**Direct `del` and `.clear()` on `net.nogoods`.** The tests mutate the nogoods list directly rather than going through an API method. This is intentional — it's the worst case. If the counter survives raw list mutation, it survives anything.

## Dependencies

**Imports:**
- `reasons_lib.api` — High-level API (`init_db`, `add_node`, `add_nogood`, `import_json`)
- `reasons_lib.Nogood` — The nogood data class (imported but not directly referenced — likely used by type context)
- `reasons_lib.network.Network` — The in-memory belief network with the `_next_nogood_id` counter
- `reasons_lib.storage.Storage` — SQLite persistence layer (`save`, `load`, `close`)
- `reasons_lib.import_beliefs.import_into_network` — Markdown import function

**Imported by:** Nothing — this is a test file.

## Flow

Each test follows the same pattern:

1. Create a `Network` (or initialize a DB via `api.init_db`)
2. Add nodes as preconditions (nogoods reference node IDs)
3. Add nogoods, optionally delete some
4. Assert `_next_nogood_id` and the generated `.id` strings
5. For persistence tests: save → close → reopen → load → assert counter survived
6. For import tests: feed external data → assert counter absorbed the max imported ID

The ID format is `nogood-NNN` (zero-padded to 3 digits). The counter `_next_nogood_id` always holds the *next* integer to use, not the last one used.

## Invariants

1. **Monotonicity.** `_next_nogood_id` never decreases — not after deletion, not after clearing, not after save/load, not after import.
2. **No ID reuse.** A nogood ID, once assigned, is never assigned again even if the original nogood is deleted.
3. **High-water mark on import.** When importing nogoods with explicit IDs (JSON or markdown), the counter must be set to `max(existing_counter, max_imported_id + 1)`.
4. **Backward compatibility.** Databases without `network_meta` must still load. If nogoods exist, the counter is derived from them; if not, it defaults to 1.

## Error Handling

This file doesn't test error paths — it's focused on the correctness of the counter invariant. There's no explicit exception testing. The `test_old_db_*` tests are the closest to error handling: they verify graceful degradation when expected schema is missing.

---

## Topics to Explore

- [function] `reasons_lib/network.py:add_nogood` — Where the monotonic counter is incremented and the `nogood-NNN` ID is formatted
- [file] `reasons_lib/storage.py` — How `network_meta` is created, saved, and loaded — the persistence side of this fix
- [function] `reasons_lib/api.py:import_json` — How JSON import scans incoming nogood IDs to bump the counter
- [diff] `issue-26-fix` — The original PR that introduced `_next_nogood_id` to see what changed in `Network` and `Storage`
- [general] `nogood-semantics` — What nogoods represent in a TMS (contradictory belief sets) and why stable IDs matter for tracking resolutions

## Beliefs

- `nogood-id-counter-is-monotonic` — `Network._next_nogood_id` only ever increases; deletion, clearing, and reload never decrease it
- `nogood-id-format-is-zero-padded` — Nogood IDs follow the format `nogood-NNN` with 3-digit zero-padded integers (e.g., `nogood-001`)
- `storage-derives-counter-from-existing-nogoods` — When loading a database without `network_meta`, `Storage.load()` derives `_next_nogood_id` from the max ID of existing nogoods rather than defaulting to 1
- `import-paths-bump-nogood-counter` — Both `import_json` and `import_into_network` update `_next_nogood_id` to be at least `max_imported_id + 1`

