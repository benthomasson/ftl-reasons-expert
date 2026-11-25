---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How (or whether) `SCHEMA_VERSION` from `reasons.metadata` is read and enforced; `Storage.save` uses `setdefault` and `_init_schema` appears not to consult it at all.

## Short answer

`SCHEMA_VERSION` from `reasons/metadata.py:11` is written into metadata on DB initialization and as a fallback during save, but it is **not validated or enforced**. There is no observed code path that reads the stored `schema_version` and compares it to `SCHEMA_VERSION`, rejects a mismatch, or triggers a versioned migration.

`Storage._init_schema` seeds `schema_version` only for a new DB. `Storage.save` uses `setdefault`, so it only supplies `SCHEMA_VERSION` when `network.meta` does not already contain a `schema_version`. If the in-memory network has an older or custom value, that value is saved instead.

## Where `SCHEMA_VERSION` is used

### 1. Definition and export metadata

- `reasons/metadata.py:11` defines `SCHEMA_VERSION = "1.0"`.
- `reasons/metadata.py:49` puts it into `build_meta(...)` output as `"schema_version": SCHEMA_VERSION`.

That makes `build_meta` a write path: it always stamps exports with the current constant. It is not a validation path.

### 2. `Storage._init_schema`

`Storage.__init__` determines `_is_new` from whether the DB path exists, opens SQLite, then calls `_init_schema`.

Inside `_init_schema`, the code:

1. reads existing table names via `sqlite_master`;
2. executes the `SCHEMA` script with `CREATE TABLE IF NOT EXISTS ...`;
3. performs structural migrations using `PRAGMA table_info(...)` and column-presence checks;
4. if `self._is_new` is true, inserts default metadata rows, including:
   - `reasons/storage.py:195`: `("schema_version", SCHEMA_VERSION),`

Important point: this insert is guarded by `if self._is_new:`. For an existing DB, `_init_schema` does **not** seed or inspect `schema_version`.

There is no code in `_init_schema` like:

```python
stored = conn.execute(
    "SELECT value FROM network_meta WHERE key = 'schema_version'"
).fetchone()
if stored and stored[0] != SCHEMA_VERSION:
    ...
```

The migrations are opportunistic structural fixes — add missing columns, populate `node_tags`/`node_sources` when those tables did not previously exist — not schema-version-gated migrations.

### 3. `Storage.save`

`Storage.save` deletes `network_meta` and later rebuilds it from `network.meta`:

- `reasons/storage.py:326`:
  ```python
  meta.setdefault("schema_version", SCHEMA_VERSION)
  ```

`setdefault` means:

- if `network.meta` already has `"schema_version"`, that value is used;
- if it does not, the current `SCHEMA_VERSION` is written.

There is no equality check against `SCHEMA_VERSION`, no warning, and no rejection. A network with `network.meta["schema_version"] = "0.9"` would save `"0.9"`.

Also, because `Storage.save` deletes `network_meta` and repopulates it only from `network.meta` plus defaults, it does **not** preserve DB-only metadata unless that metadata was already loaded into `network.meta`. So saving a fresh `Network()` over an existing DB can overwrite the stored `schema_version` with the current constant.

### 4. Load and import/export paths

The lowercase `schema_version` appears as an ordinary metadata key in several places:

- `reasons/api.py:2127`: loops over `("schema_version", "project_name", "created_at")`.
- `reasons/import_beliefs.py:183`: same pattern.
- `reasons/pg.py:2250` and `reasons/pg.py:2398`: same pattern in PostgreSQL-related metadata handling.
- `reasons/export_card.py:68` and `reasons/export_markdown.py:25`: export the field into frontmatter.

These are copying/serialization paths, not enforcement paths. They treat `schema_version` as a normal metadata field.

The PostgreSQL path has the same default-seeding pattern:

- `reasons/pg.py:129` imports `SCHEMA_VERSION`;
- `reasons/pg.py:151` inserts `("schema_version", SCHEMA_VERSION)` during initialization.

The observed evidence does not show any comparison or migration decision based on that value.

### 5. Tests confirm defaulting/preservation, not enforcement

Tests assert that new metadata contains the constant and that it round-trips:

- `tests/test_metadata.py:192`: new SQLite DB has `rows["schema_version"] == SCHEMA_VERSION`.
- `tests/test_metadata_extended.py:175`: similar DB assertion.
- `tests/test_metadata_extended.py:201`: loaded network metadata still has `schema_version == SCHEMA_VERSION`.

What is not present in the observed tests is a test for:

- refusing to open a DB with a different `schema_version`;
- migrating from an older `schema_version`;
- warning or failing when `network.meta["schema_version"] != SCHEMA_VERSION`.

That matches the code behavior: the version is written as metadata but not used as a compatibility gate.

## Bottom line

`SCHEMA_VERSION` is currently a **write-time default and export stamp**, not an enforced schema contract.

`Storage._init_schema` does not consult it for existing DBs. `Storage.save` only fills it in when missing via `setdefault`. A stored mismatched `schema_version` would be preserved — or potentially overwritten with the current constant if the network metadata is not loaded first — without any validation.

## Topics to Explore

- [file] `reasons/storage.py` — Read the full `Storage.load` implementation past the returned excerpt to confirm how `network_meta` is loaded generically into `Network.meta`.
- [function] `reasons/storage.py:_init_schema` — Verify that migrations are column/table-existence driven rather than version driven.
- [file] `reasons/pg.py` — Compare the PostgreSQL storage path for the same defaulting-without-validation pattern.
- [file] `reasons/api.py` — Inspect the metadata import/export loops around line 2127 to see how `schema_version` is copied between sources.
- [general] `schema-migration-policy` — Decide whether the project should reject unknown schema versions, auto-migrate, or continue treating `schema_version` as informational metadata.

## Beliefs

- `schema-version-is-write-default-only` — `SCHEMA_VERSION` is written to new DBs and missing metadata, but no observed code compares it to a stored value.
- `init-schema-does-not-read-schema-version` — `Storage._init_schema` seeds `network_meta.schema_version` only when `_is_new` is true and does not inspect existing schema-version metadata.
- `storage-save-preserves-network-meta-schema-version` — `Storage.save` uses `meta.setdefault("schema_version", SCHEMA_VERSION)`, so an existing `network.meta["schema_version"]` overrides the constant.
- `storage-save-overwrites-db-only-network-meta` — `Storage.save` deletes `network_meta` and repopulates it from `network.meta` plus defaults, so DB-only metadata is not preserved unless loaded first.
- `schema-version-mismatch-not-rejected` — No observed storage, API, import, or PostgreSQL path raises or warns when stored `schema_version` differs from `SCHEMA_VERSION`.
