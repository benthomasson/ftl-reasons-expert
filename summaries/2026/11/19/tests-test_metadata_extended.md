---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_metadata_extended.py

# `tests/test_metadata_extended.py`

## Purpose

This is a pytest contract/integration test module for the metadata layer of the `reasons` package. It extends `tests/test_metadata.py` by covering edge cases and cross-format behavior:

- building metadata directly with `build_meta`
- JSON export/import/re-export round-trips
- SQLite `network_meta` persistence
- Markdown frontmatter export/parsing/import
- `Storage.load()` / `Storage.save()` metadata persistence
- backward compatibility for JSON without `meta` and Markdown without frontmatter

It is not production code and is not imported by application modules. Pytest discovers it and runs each class/test method. It specifically imports from `reasons`, not the mirrored `reasons_lib` package.

Notable maintenance detail: the module docstring claims coverage for “Metadata via CLI” and “updated_at changes between saves,” but this file contains no CLI test and no direct test that `updated_at` changes between saves. Also, `pytest` and `import_into_network` are imported but not used in the shown file.

## Key Components

### `TestBuildMetaEdgeCases`

Exercises `reasons.metadata.build_meta`, `SCHEMA_VERSION`, and the private `_get_generator`.

- `test_node_count_zero`: `build_meta("proj", node_count=0)` preserves `node_count == 0`.
- `test_large_node_count`: large counts are preserved.
- `test_created_at_not_overwritten_when_provided`: a caller-supplied `created_at` is retained, while `updated_at` is generated independently and differs from the provided timestamp.
- `test_updated_at_is_iso_format`: `updated_at` contains `T` and ends with `+00:00`.
- `test_schema_version_constant`: `SCHEMA_VERSION == "1.0"`.
- `test_generator_format`: `_get_generator()` returns exactly `ftl-reasons/<version>`; it starts with `ftl-reasons/`, splits into two parts, and the version is non-empty.

This class couples tests to a private helper (`_get_generator`), so renaming that helper will break tests.

### `TestJsonRoundTripMeta`

Tests the JSON export/import contract through `reasons.api`.

- `test_full_roundtrip_preserves_meta`:
  - create DB `db1` with `project_name="round-trip-test"`
  - add premise `a` and derived node `b` with `sl="a"`
  - export via `api.export_network(db_path=db1)`
  - write JSON to disk
  - create DB `db2` without explicit project name
  - import via `api.import_json(json_file, db_path=db2)`
  - re-export and assert `schema_version`, `project_name`, and `node_count == 2`
  - This proves imported JSON metadata overrides destination DB defaults.
- `test_node_count_matches_exported_nodes`: after adding 5 nodes, `meta["node_count"] == len(data["nodes"])`.
- `test_node_count_after_retraction`: retracted nodes still count because they remain part of the export; the test asserts `node_count == len(data["nodes"])`.
- `test_export_empty_network`: empty DB exports `schema_version == SCHEMA_VERSION`, `node_count == 0`, and default `project_name == "empty"` derived from `empty.db`.
- `test_import_json_without_meta_uses_defaults`: imports a legacy JSON shape with `nodes` as a dict keyed by node ID and no `meta` key. Re-export has default schema metadata and correct `node_count == 1`.
- `test_meta_generator_in_export`: exported metadata includes a `generator` starting with `ftl-reasons/`.

### `TestSqliteMetaExtended`

Uses direct SQLite queries against the `network_meta` table, so these tests are white-box with respect to persistence schema.

- `test_created_at_preserved_across_saves`: reads `created_at` from `network_meta`, adds nodes, re-reads, and asserts `created_at` did not change.
- `test_schema_version_in_db`: asserts `network_meta["schema_version"] == SCHEMA_VERSION`.
- `test_default_project_name_from_filename`: `my-beliefs.db` produces default `project_name == "my-beliefs"` using the filename stem.
- `test_meta_survives_load_save_cycle`: uses `Storage` directly, mutates `net.meta`, saves, reopens, and asserts `project_name`, `created_at`, and `schema_version` survive.

### `TestMarkdownExportMetaExtended`

Tests `reasons.export_markdown.export_markdown`.

- `test_frontmatter_is_first_block`: first line is `---`.
- `test_frontmatter_ends_with_separator`: at least two lines strip to `---`, and the first is line 0.
- `test_frontmatter_node_count_matches`: exported Markdown contains `node_count: 3`.
- `test_empty_project_name_handled`: Markdown contains a `project_name:` field. The test does not assert the value is empty; it only verifies the field is emitted.
- `test_frontmatter_schema_version_quoted`: Markdown contains exactly `schema_version: "1.0"`.

### `TestStripFrontmatterEdgeCases`

Tests `reasons.import_beliefs.strip_frontmatter`, which parses simple frontmatter into `(body, frontmatter_dict)`.

- `test_incomplete_frontmatter_no_closing`: if there is no closing `---`, return the original text unchanged and `{}`.
- `test_empty_value`: `project_name:` becomes `""`.
- `test_single_quoted_value`: `'my-project'` becomes `my-project`.
- `test_frontmatter_with_numeric_value`: `node_count: 42` becomes the string `"42"`; this is not typed YAML parsing.
- `test_body_preserved_after_strip`: body retains headings and paragraphs and no longer contains the `---` lines.

### `TestMarkdownRoundTripExtended`

Tests end-to-end Markdown export/import through the public API.

- `test_project_name_roundtrip`: export Markdown from a DB with `project_name="my-proj"`, import into a second DB, re-export and assert the project name survives.
- `test_schema_version_roundtrip`: same flow, asserting `schema_version == SCHEMA_VERSION`.
- `test_import_markdown_without_frontmatter_preserves_existing_meta`: imports old-style Markdown without frontmatter into an existing DB and asserts the existing `project_name` is not clobbered.

## Patterns

- **Arrange-Act-Assert** test structure throughout.
- **Test grouping by concern**, using classes for metadata building, JSON, SQLite, Markdown export, frontmatter stripping, and Markdown round-trips.
- **Filesystem isolation** via pytest’s `tmp_path`; each test writes local `.db`, `.json`, and `.md` files.
- **Round-trip testing**: export → serialize/write → import → re-export → assert.
- **Backward-compatibility testing**: legacy JSON without `meta`, old Markdown without frontmatter.
- **White-box testing**: direct `sqlite3` queries against `network_meta`, direct use of `_get_generator`, and direct calls to `strip_frontmatter`.
- **Real dependencies, no mocks**: uses real SQLite databases and real files.
- **Public API plus internal helpers**: most tests use `reasons.api`, but some reach into `reasons.metadata`, `reasons.import_beliefs`, and `reasons.storage`.

## Dependencies

### What this file imports

- `json` — serializes export dicts in JSON round-trip tests.
- `sqlite3` — direct inspection of `network_meta`.
- `pathlib.Path` — writes JSON/Markdown fixture files.
- `pytest` — imported but not directly used in the shown code.
- `reasons.api` — high-level operations: `init_db`, `add_node`, `retract_node`, `export_network`, `import_json`, `import_beliefs`, `export_markdown`.
- `reasons.metadata.build_meta`, `SCHEMA_VERSION`, `_get_generator` — metadata construction and constants.
- `reasons.network.Network` — used for direct Markdown export tests.
- `reasons.export_markdown.export_markdown` — direct Markdown export.
- `reasons.import_beliefs.strip_frontmatter`, `import_into_network` — `strip_frontmatter` is tested; `import_into_network` is imported but unused.
- `reasons.storage.Storage` — load/save metadata cycle test.

### What depends on it

No production code depends on this file. Pytest discovers and executes it. It indirectly enforces contracts of:

- `reasons.metadata`
- `reasons.storage`
- `reasons.export_markdown`
- `reasons.import_beliefs`
- `reasons.api`

## Flow

Typical execution paths:

1. **Direct metadata tests**
   - call `build_meta(...)`
   - inspect returned dict keys and values
   - assert schema version, generator format, timestamp behavior

2. **JSON round-trip**
   - `api.init_db(db1)`
   - mutate nodes via `api.add_node` / `api.retract_node`
   - `api.export_network(db1)` → Python dict
   - `json.dumps` → file
   - `api.init_db(db2)`
   - `api.import_json(file, db2)`
   - `api.export_network(db2)` → assert metadata survived

3. **SQLite metadata**
   - open `sqlite3.connect(db)`
   - query `SELECT key, value FROM network_meta`
   - close connection
   - mutate via API or `Storage`
   - reopen and re-query to assert persistence

4. **Markdown export**
   - build a `Network()` or use API DB
   - call `export_markdown`
   - inspect string line-by-line and via substring assertions

5. **Frontmatter parsing**
   - call `strip_frontmatter(text)`
   - receive `(body, fm)`
   - assert fallback and normalization behavior

6. **Markdown import**
   - export Markdown from `db1`
   - write to `.md`
   - `api.import_beliefs(file, db2)`
   - re-export from `db2` and assert metadata survived

7. **Storage load/save**
   - `Storage(db).load()`
   - mutate `net.meta`
   - `store.save(net)`
   - reopen and assert metadata persisted

## Invariants

- `SCHEMA_VERSION` must remain `"1.0"`.
- `_get_generator()` must return `ftl-reasons/<non-empty-version>` with exactly one slash.
- `updated_at` is ISO-8601 UTC formatted: contains `T` and ends with `+00:00`.
- When `created_at` is supplied to `build_meta`, it must be preserved.
- SQLite `network_meta` must contain `schema_version`, `project_name`, and `created_at`.
- Default `project_name` for `init_db` without an explicit name is the database filename stem, e.g. `empty.db` → `empty`, `my-beliefs.db` → `my-beliefs`.
- `created_at` must not change when adding nodes or saving the network.
- Exported metadata `node_count` must equal `len(data["nodes"])`.
- Retracted nodes remain part of the exported nodes and therefore count toward `node_count`.
- JSON import without metadata must still produce default metadata on re-export.
- JSON import with metadata must preserve imported `project_name` and `schema_version`.
- Markdown frontmatter must start at the first line and be delimited by `---`.
- Markdown `schema_version` must be quoted: `schema_version: "1.0"`.
- `strip_frontmatter` treats all parsed values as strings.
- Incomplete frontmatter must not raise; it returns the original text and `{}`.
- Importing Markdown without frontmatter must not overwrite existing metadata.
- `Storage.save()` / `Storage.load()` must preserve the `meta` dict.

## Error Handling

There is no explicit `try`/`except` in this test file. The suite encodes “error handling” as expected graceful-degradation behavior:

- Incomplete frontmatter is not an exception; `strip_frontmatter` returns the original body and empty metadata.
- Legacy JSON without `meta` imports successfully using defaults.
- Old Markdown without frontmatter imports successfully without clobbering existing metadata.

If production code raises unexpectedly, pytest will fail the test. SQLite connections are closed explicitly with `conn.close()` after queries, though not via context managers.

## Topics to Explore

- [file] `reasons/metadata.py` — Implements `build_meta`, `SCHEMA_VERSION`, and `_get_generator`; inspect exact timestamp and generator behavior.
- [file] `reasons/import_beliefs.py` — Contains `strip_frontmatter` and Markdown import logic; verify parsing and metadata merge rules.
- [file] `reasons/storage.py` — Persists `network_meta` and implements `Storage.load()` / `save()`; confirm created_at/updated_at save semantics.
- [file] `reasons/export_markdown.py` — Produces the frontmatter format asserted here, including quoting and field ordering.
- [general] `metadata-schema-versioning` — Explore how `schema_version == "1.0"` is used for compatibility and whether future migrations are planned.

## Beliefs

- `metadata-schema-version-is-1-0` — `SCHEMA_VERSION` is asserted equal to `"1.0"` and is stored in SQLite `network_meta` and exported metadata.
- `generator-format-requires-ftl-prefix` — `_get_generator()` returns `ftl-reasons/<version>` with exactly two slash-separated parts.
- `node-count-tracks-exported-nodes` — `export_network` metadata `node_count` equals `len(data["nodes"])`, including retracted nodes still present in the export.
- `default-project-name-derives-from-db-stem` — When `init_db` receives no project name, `project_name` equals the database filename stem, e.g. `my-beliefs.db` → `my-beliefs`.
- `missing-frontmatter-preserves-existing-meta` — Importing Markdown without frontmatter leaves existing `project_name` metadata unchanged.
