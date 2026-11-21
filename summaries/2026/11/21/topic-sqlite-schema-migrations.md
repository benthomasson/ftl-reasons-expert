---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Worth checking because this test only covers fresh-database round trips, not schema upgrades or backward compatibility.

## What "worth checking" means here

The note is about a specific axis of coverage that the round-trip test suite does **not** exercise: **legacy databases being opened and upgraded by newer code**. Almost every test in `tests/test_roundtrip.py` starts from a freshly-created SQLite file, so migration branches in the storage layer are never hit.

### Why these tests never exercise upgrades

The shared fixture at the top of `tests/test_roundtrip.py`:

```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "test.db")
    api.init_db(db_path=db_path)
    return db_path
```

`tmp_path` is a per-test empty directory and `test.db` doesn't exist, so `api.init_db` always calls `Storage(...)` with `self._is_new = True` (`reasons/storage.py:Storage.__init__`, near the `self._is_new = not self.db_path.exists()` line). Every test in the file — `TestSourceUrlRoundTrip`, `TestAcceptedPrRoundTrip`, `TestFullRoundTrip.test_all_fields_survive`, `test_stale_node_round_trip`, `test_unless_round_trip`, `test_double_round_trip_stability`, `test_frontmatter_round_trip` — is therefore a **fresh-schema** round trip. The same is true of the `tmp_path`-based tests named in `find_roundtrip_usages` (`tests/test_metadata_extended.py:test_full_roundtrip_preserves_meta`, `test_schema_version_roundtrip`, `tests/test_api.py:test_import_roundtrips_timestamps`, etc.).

In other words, the whole suite asserts *serialization fidelity* (add → export → import → same fields), not *schema evolution fidelity* (old file → new code → safe read/upgrade).

## What the fresh-db path skips in `reasons/storage.py`

`Storage._init_schema` is where the upgrade logic lives, but the fresh-db fixture bypasses it because `SCHEMA` is `CREATE TABLE IF NOT EXISTS` with the *current* columns already present. That means the following branches never run in `test_roundtrip.py`:

- **Column adds** — `ALTER TABLE nodes ADD COLUMN source_url …`, the loop for `created_at/updated_at/reviewed_at/verified_at/retracted_at`, `supporting_justification`, `text_hash`, and `ALTER TABLE justifications ADD COLUMN content_hash …`.
- **Data backfills** — the `if "node_tags" not in pre_tables:` block that walks `SELECT id, metadata_json FROM nodes` and inserts `access:<tag>` rows, and the `if "node_sources" not in pre_tables:` block that reconstructs `node_sources` rows from legacy `nodes.source`, `nodes.source_url`, `nodes.source_hash`, and `metadata_json` (pinned_sha / pinned_lines). These are exactly the compatibility shims for old databases.
- **`pre_tables` snapshot** — computed via `SELECT name FROM sqlite_master …` *before* `executescript(SCHEMA)`; on a fresh DB this set is empty and every `if "…" not in pre_tables` is trivially true, but the inserts find no rows to backfill, so any bug in the backfill SQL is invisible.

The note also covers a subtler gap: **there is no version-gated migration in `_init_schema`**. It never reads `network_meta.schema_version`. `Storage.save` only does `meta.setdefault("schema_version", SCHEMA_VERSION)` — i.e., it silently stamps the *current* version onto whatever it writes. So there's no code path that says "refuse to open version N" or "run migration from N → N+1"; the only compatibility mechanism is "add the column if `PRAGMA table_info` doesn't list it." Tests that would catch this (old file with `schema_version: 1`, then open with current code) do not exist in the observations, even though `tests/test_metadata_extended.py:test_schema_version_roundtrip` sounds like it might — it's still a `tmp_path` fresh DB.

## The PostgreSQL analogue

`reasons/pg.py:PgApi.init_db` contains equivalent migration logic:

- `SELECT column_name FROM information_schema.columns WHERE table_name = 'rms_nodes' AND column_name = 'source_url'` → `ALTER TABLE rms_nodes ADD COLUMN source_url …` if absent.
- A loop over `("updated_at", "reviewed_at", "verified_at", "retracted_at")` doing the same `information_schema` check + `ALTER TABLE`.

On a fresh database these `information_schema` probes always return the column (having just been created by `SCHEMA`), so the `ALTER` branches are dead in fresh-db tests. No test in the observations opens a `rms_nodes` table created by an older version and asserts the `ALTER` fires and backfills correctly.

## Why this is "worth checking" rather than a bug

The round-trip tests themselves are *correct* — they check export/import field fidelity. The risk is **coverage illusion**: `tests/test_roundtrip.py`, `tests/test_metadata_extended.py:test_full_roundtrip_preserves_meta`, and `tests/test_api.py:test_import_roundtrips_timestamps` all look like they cover persistence comprehensively, but they all sit on the same fresh-`Storage` fixture. A regression in the `ALTER TABLE` / backfill code in `Storage._init_schema` would ship green. Notably, the same file already documents one pre-existing fidelity gap (`test_nogoods_export_format` at `tests/test_roundtrip.py` around line 227: *"parse_nogoods expects 'nogood-N: label' format but export writes 'nogood-NNN' without colon… pre-existing gap"*), which shows the file is already where such gaps get noticed — but schema-upgrade coverage isn't there yet.

## What's missing from these observations

- **The rest of `reasons/storage.py`** — `load()` starts at the end of the excerpt (line ~350 of 482). Whether `load()` tolerates missing columns / old `metadata_json` shapes is not visible.
- **Any legacy-fixture test** — none shown constructs an old SQLite file (hand-rolled `CREATE TABLE` or checked-in fixture) and then opens it with `Storage`. No such file appears in `find_roundtrip_usages`.
- **`find_schema_migration_symbols` returned 0 matches**, but `ALTER TABLE`/`CREATE TABLE` clearly exist in `storage.py` and `pg.py`. That search result looks like a tooling artifact (pattern/scope mismatch), so don't read it as "there is no migration code."
- **CI configuration** — no way to tell from these observations whether migration tests exist elsewhere (e.g., in a `tests/migrations/` directory not searched).

## Topics to Explore

- [function] `reasons/storage.py:Storage._init_schema` — The only place SQLite upgrade/migration happens; trace the `pre_tables` snapshot and every `ALTER TABLE` / backfill loop to see exactly what a fresh-db test skips.
- [function] `reasons/pg.py:PgApi.init_db` — PG's equivalent `information_schema.columns` + `ALTER TABLE` migration branches, also skipped by fresh-db tests.
- [general] `schema-version-migration-policy` — How (or whether) `SCHEMA_VERSION` from `reasons.metadata` is read and enforced; `Storage.save` uses `setdefault` and `_init_schema` appears not to consult it at all.
- [file] `reasons/storage.py` — The `load()` method (line ~350+) is cut off in these observations; verify it handles rows produced by older schemas after the `ALTER`-only upgrade.
- [file] `tests/test_roundtrip.py` — Read the fixture and `test_nogoods_export_format` side-by-side; the file already documents a known fidelity gap, and the fixture is the reason the migration branch never fires.

## Beliefs

- `roundtrip-tests-use-fresh-database-fixture` — The `db` fixture in `tests/test_roundtrip.py` creates a new file under `tmp_path` and calls `api.init_db`, so `Storage._is_new` is always `True` for these tests.
- `storage-init-schema-adds-missing-columns` — `Storage._init_schema` uses `PRAGMA table_info` followed by `ALTER TABLE … ADD COLUMN` for `source_url`, timestamps, `supporting_justification`, `text_hash`, and `justifications.content_hash`.
- `storage-init-schema-backfills-node-tags-and-sources` — When `node_tags` / `node_sources` are absent from `pre_tables`, `_init_schema` repopulates them from `nodes.metadata_json` and `nodes.source*`; fresh-db tests have no rows to backfill, so this SQL is unexercised.
- `schema-version-is-not-migration-driver` — `_init_schema` never reads `network_meta.schema_version`, and `Storage.save` only `setdefault`s the current `SCHEMA_VERSION`; there is no version-gated migration path visible in the observations.
- `pg-init-db-also-has-untested-alter-branches` — `PgApi.init_db` performs `information_schema.columns` checks and `ALTER TABLE rms_nodes ADD COLUMN` for `source_url` and the timestamp columns, which fresh-database tests skip.
