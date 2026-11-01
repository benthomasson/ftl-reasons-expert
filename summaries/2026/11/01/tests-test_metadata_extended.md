---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_metadata_extended.py

# `tests/test_metadata_extended.py` — Extended Metadata Tests Across Storage Formats

## 1. Purpose

This file is the **second-tier test suite** for the metadata subsystem of `ftl-reasons`. Where `test_metadata.py` covers the happy-path basics (building a meta dict, basic SQLite persistence), this file owns the **edge cases, cross-format round-trips, and format-specific invariants** that are easy to break but hard to notice until a user migrates data between formats.

Specifically it verifies that the `meta` block — a small dictionary of bookkeeping fields (`project_name`, `schema_version`, `created_at`, `updated_at`, `node_count`, `generator`) — survives the full lifecycle: JSON export → JSON import, SQLite load → save, and Markdown frontmatter export → Markdown import. It also pins down the **parsing behavior** of `strip_frontmatter` against malformed or partial input.

## 2. Key Components

### Test Classes (each maps to a subsystem)

| Class | What it exercises |
|---|---|
| `TestBuildMetaEdgeCases` | The `build_meta()` factory in `reasons/metadata.py` — boundary values, timestamp semantics, generator format |
| `TestJsonRoundTripMeta` | `api.export_network` / `api.import_json` — meta survives a full JSON file round-trip |
| `TestSqliteMetaExtended` | Direct `sqlite3` inspection of the `network_meta` table; `Storage.load()` / `Storage.save()` |
| `TestMarkdownExportMetaExtended` | `export_markdown()` — frontmatter structure, ordering, quoting |
| `TestStripFrontmatterEdgeCases` | `strip_frontmatter()` parser — malformed, quoted, empty, numeric values |
| `TestMarkdownRoundTripExtended` | `api.export_markdown` → `api.import_beliefs` — meta propagates through the Markdown path |

### Key Functions Under Test (from `reasons/`)

- **`build_meta(project_name, node_count, created_at, …)`** (`reasons/metadata.py`)
  - Contract: returns a dict with keys `project_name`, `schema_version`, `created_at`, `updated_at`, `node_count`, `generator`.
  - `created_at` defaults to *now* unless explicitly provided; `updated_at` is *always* set to now (test at line `test_created_at_not_overwritten_when_provided` confirms they diverge).
  - `SCHEMA_VERSION` is a module-level constant pinned to `"1.0"`.
  - `_get_generator()` returns `"ftl-reasons/<version>"` (two parts, slash-separated).

- **`strip_frontmatter(text)`** (`reasons/import_beliefs.py`)
  - Contract: returns `(body, frontmatter_dict)`.
  - If no closing `---` is found, returns `(original_text, {})` — a graceful no-op, not an exception.
  - Values are returned as **strings** even for numeric-looking keys (e.g. `"node_count": "42"`), and single-quoted values have their quotes stripped.

- **`export_markdown(net)`** (`reasons/export_markdown.py`)
  - Produces a Markdown document whose first block is YAML frontmatter delimited by `---` on line 0 and a closing `---`.
  - `schema_version` is double-quoted in the output (`schema_version: "1.0"`).

- **`Storage(db_path)`** (`reasons/storage.py`)
  - `load()` → `Network` object with a `.meta` dict populated from the `network_meta` table.
  - `save(net)` → persists `net.meta` back; must not clobber `created_at`.

### Fixtures & Helpers

- **`tmp_path`** (pytest built-in): every test that touches the filesystem gets an isolated temp directory. No shared state between tests.
- Direct **`sqlite3.connect(db)`** calls are used in `TestSqliteMetaExtended` to bypass the ORM/API layer and assert on the raw `network_meta` table (`SELECT key, value FROM network_meta`).

## 3. Patterns

- **Round-trip testing**: The dominant pattern is *write → serialize → deserialize → re-serialize → compare*. Three variants are tested (JSON, SQLite, Markdown). This is the standard way to catch lossy serialization bugs.

  ```python
  data1 = api.export_network(db_path=db1)   # serialize
  api.import_json(json_file, db_path=db2)   # deserialize
  data2 = api.export_network(db_path=db2)   # re-serialize
  assert data2["meta"]["project_name"] == "round-trip-test"  # compare
  ```

- **Direct DB inspection**: `TestSqliteMetaExtended` opens a raw `sqlite3` connection to read the `network_meta` table. This bypasses the API layer, ensuring the *storage contract* (column names, value types) is stable, not just the *API contract*.

- **Edge-case enumeration**: `TestStripFrontmatterEdgeCases` enumerates parser failure modes (no closing `---`, empty value, single-quoted value, numeric value) as separate test methods rather than a parameterized test. This makes each failure mode independently debuggable.

- **Invariant pinning**: Tests like `test_created_at_preserved_across_saves` and `test_node_count_matches_exported_nodes` encode invariants that the production code *must* uphold. They read as executable specifications.

- **Legacy compatibility**: `test_import_json_without_meta_uses_defaults` and `test_import_markdown_without_frontmatter_preserves_existing_meta` ensure that old-format files (no `meta` key, no frontmatter) don't crash the importer and don't clobber existing metadata.

## 4. Dependencies

### Imports (what this file depends on)

| Import | Source | Why |
|---|---|---|
| `json`, `sqlite3`, `Path` | stdlib | File I/O and direct DB access |
| `pytest` | third-party | Test runner, `tmp_path` fixture |
| `reasons.api` | local | High-level API: `init_db`, `add_node`, `export_network`, `import_json`, `import_beliefs`, `export_markdown`, `retract_node` |
| `reasons.metadata` | local | `build_meta`, `SCHEMA_VERSION`, `_get_generator` — the metadata factory and constants |
| `reasons.network` | local | `Network` — the in-memory graph object (used directly in Markdown export tests) |
| `reasons.export_markdown` | local | `export_markdown` — tested in isolation without going through `api` |
| `reasons.import_beliefs` | local | `strip_frontmatter` — the frontmatter parser under test |
| `reasons.storage` | local | `Storage` — the SQLite persistence layer |

### What depends on this file

Nothing imports this file. It is a **leaf test module** discovered by pytest via the `tests/` package. Its `conftest.py` (sibling) likely provides shared fixtures, but this file uses only the built-in `tmp_path`.

## 5. Flow

Each test follows a short, linear flow. The most complex flow is the **JSON round-trip** (`test_full_roundtrip_preserves_meta`):

```
1. init_db(db1, project_name="round-trip-test")     → creates SQLite, writes meta
2. add_node("a", …) / add_node("b", sl="a")         → populates graph
3. export_network(db1)                               → dict with "meta", "nodes", …
4. json.dumps → write to file                        → serialize
5. init_db(db2)                                      → fresh empty DB
6. import_json(file, db2)                            → deserialize, rebuild graph + meta
7. export_network(db2)                               → dict from new DB
8. assert meta fields match                          → verify no data lost
```

The **SQLite meta tests** are shorter:

```
1. init_db → read network_meta table (sqlite3)
2. add_node × N
3. read network_meta table again
4. assert created_at unchanged, schema_version stable
```

The **frontmatter parser tests** are pure-function:

```
input string → strip_frontmatter() → (body, dict) → assert on both
```

No async, no network calls, no mocking. Everything is synchronous file I/O against `tmp_path`.

## 6. Invariants

The tests collectively enforce these rules:

1. **`created_at` is immutable after initial write.** Once `init_db` writes it, subsequent `add_node` / `save` calls must not overwrite it. (`test_created_at_preserved_across_saves`)
2. **`updated_at` is always refreshed to the current time** and is ISO-8601 with a `+00:00` suffix. (`test_updated_at_is_iso_format`)
3. **`node_count` equals the number of nodes in the export**, including retracted ones (they're still in the JSON). (`test_node_count_after_retraction`)
4. **`schema_version` is the constant `"1.0"`** everywhere — DB, JSON export, Markdown frontmatter. (`test_schema_version_constant`, `test_schema_version_in_db`)
5. **`project_name` defaults to the DB filename stem** (e.g. `my-beliefs.db` → `"my-beliefs"`) when not explicitly provided. (`test_default_project_name_from_filename`)
6. **Frontmatter must be the first block** in any Markdown export, delimited by `---` on line 0. (`test_frontmatter_is_first_block`)
7. **`strip_frontmatter` is total**: it never raises; malformed input yields `({}, original_text)`. (`test_incomplete_frontmatter_no_closing`)
8. **Importing legacy data (no meta, no frontmatter) must not crash and must not destroy existing meta.** (`test_import_json_without_meta_uses_defaults`, `test_import_markdown_without_frontmatter_preserves_existing_meta`)

## 7. Error Handling

This is a test file, so "error handling" here means **verifying that the production code handles errors gracefully**:

- **Malformed frontmatter**: `test_incomplete_frontmatter_no_closing` asserts that `strip_frontmatter` returns the *original text unchanged* and an *empty dict* rather than raising a `ValueError` or `KeyError`. This is a critical contract because `import_beliefs` calls `strip_frontmatter` on user-supplied files that may be hand-edited.

- **Missing `meta` in JSON**: `test_import_json_without_meta_uses_defaults` constructs a legacy JSON payload with no `"meta"` key. The test asserts the import succeeds and the re-exported meta has valid defaults. The production code in `api.import_json` / `import_into_network` must use `.get("meta", {})` or equivalent.

- **Empty values in frontmatter**: `test_empty_value` (`project_name:` with no value) expects the parser to set it to `""`, not `None` or crash.

- **No exception-based assertions**: Not a single test in this file uses `pytest.raises`. Every "error" case is tested as a *graceful-degradation* path, confirming the library's philosophy of being a data layer that never throws on well-formed-but-unexpected input.

---

## Topics to Explore

- [file] `reasons/metadata.py` — The production code under test: `build_meta`, `SCHEMA_VERSION`, `_get_generator`, and the meta-dict schema definition. Understanding the exact field contracts and defaulting logic is prerequisite to maintaining these tests.
- [function] `reasons/import_beliefs.py:strip_frontmatter` — The frontmatter parser that `TestStripFrontmatterEdgeCases` exercises. Its behavior on edge cases (single quotes, empty values, missing closers) is a user-facing contract for anyone importing hand-written `.md` belief files.
- [file] `tests/test_metadata.py` — The first-tier metadata tests. Reading both files together reveals the split: which invariants are "basic" vs "extended," and whether any gap exists between the two suites.
- [file] `reasons/storage.py` — The `Storage` class that bridges `Network.meta` and the `network_meta` SQLite table. The load/save cycle tested here is the persistence contract for all metadata.
- [general] `legacy-import-compatibility` — The two "legacy" tests (JSON without meta, Markdown without frontmatter) encode backward-compatibility rules. Understanding the full set of supported input formats (and which fields are optional) is essential before changing the import path.

## Beliefs

- `created-at-is-immutable-after-init` — `build_meta` sets `created_at` to the current time only when no explicit value is provided, and `Storage.save` / `api.add_node` must never overwrite it on subsequent writes; `test_created_at_preserved_across_saves` enforces this across multiple node additions.
- `strip-frontmatter-never-raises` — `strip_frontmatter` is a total function: for any input string (including missing closing `---` or empty values), it returns a `(str, dict)` tuple without raising; malformed input degrades to `(original_text, {})`.
- `node-count-includes-retracted` — `meta["node_count"]` reflects all nodes present in the exported JSON (including those with `truth_value == "OUT"` / retracted), not just active ones; `test_node_count_after_retraction` pins this.
- `project-name-derives-from-filename` — When `init_db` is called without an explicit `project_name`, the default is the DB filename with the extension stripped (e.g. `"my-beliefs.db"` → `"my-beliefs"`), as asserted by `test_default_project_name_from_filename`.
- `schema-version-pinned-to-1-0` — `SCHEMA_VERSION` is the string `"1.0"` in every serialization surface (SQLite `network_meta` table, JSON export, Markdown frontmatter), and all round-trip tests assert equality against this single constant.
