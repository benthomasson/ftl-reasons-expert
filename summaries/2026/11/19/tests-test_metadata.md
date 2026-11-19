---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_metadata.py

# `tests/test_metadata.py`

## 1. Purpose

This is a **cross-format metadata contract test suite**. It verifies that the metadata that describes a belief network — schema version, project name, generator string, timestamps, node count — is **produced consistently** by every persistence surface the project offers, and **survives every round trip**:

- SQLite storage (`network_meta` table)
- JSON export/import (`api.export_network` / `api.import_json`)
- Markdown export/import (YAML frontmatter produced by `export_markdown`, parsed by `strip_frontmatter`)

It owns the **backward-compatibility guarantees** (old JSON without `meta`, old `beliefs.md` without frontmatter) and the **immutability invariant** that `created_at` never changes once set.

Because metadata is a stable public contract (other tools consume exported JSON/YAML), this file effectively locks the schema version and default value rules in place.

---

## 2. Key Components

### Test classes (all pytest-style, grouped by concern)

| Class | Responsibility |
|---|---|
| `TestBuildMeta` | Unit tests for `build_meta(...)` — the pure constructor of the metadata dict. |
| `TestJsonExportMeta` | Integration: `api.export_network` must inject a `meta` key with `schema_version`, `node_count`, `generator`, `project_name`. |
| `TestJsonImportMeta` | Integration: `api.import_json` round-trips `project_name` **and** tolerates legacy JSON with no `meta` key. |
| `TestMarkdownExportMeta` | `export_markdown` must begin with `---\n` frontmatter containing `schema_version`, `node_count`, `generator`, optional `project_name`. |
| `TestStripFrontmatter` | Unit tests for the frontmatter parser (`strip_frontmatter`). |
| `TestMarkdownRoundTrip` | End-to-end: markdown export → import preserves belief count; legacy markdown without frontmatter still imports. |
| `TestSqliteMeta` | Direct inspection of the `network_meta` key/value table via `sqlite3`. |

### Exercised production symbols

- **`reasons.metadata.build_meta(project_name="", node_count=0, created_at=None, ...) -> dict`**
  Contract (from tests):
  - returns `schema_version` = `SCHEMA_VERSION` (`"1.0"`)
  - returns `project_name` (default `""`)
  - returns `node_count` (int)
  - returns `generator` containing `"ftl-reasons/"`
  - returns both `created_at` and `updated_at` (ISO-8601 strings)
  - if `created_at` is supplied, it is preserved verbatim
- **`reasons.metadata.SCHEMA_VERSION`** — constant, expected `"1.0"`.
- **`reasons.api.init_db(db_path, project_name=None)`** — creates the SQLite DB and populates `network_meta`; **default project name is the DB filename stem** (`"myproject.db"` → `"myproject"`, `"test.db"` → `"test"`).
- **`reasons.api.add_node(label, text, ...)`** — mutates the network; must **not** overwrite `created_at`.
- **`reasons.api.export_network(db_path) -> dict`** — dict with `nodes`, `nogoods`, `repos`, and a new `meta` key.
- **`reasons.api.import_json(path, db_path)`** — must read `meta.project_name` when present and skip gracefully when absent.
- **`reasons.network.Network`** — in-memory model; has `.add_node(...)` and a `.meta` dict.
- **`reasons.export_markdown.export_markdown(net) -> str`** — emits `---`-delimited YAML frontmatter then body.
- **`reasons.import_beliefs.strip_frontmatter(text) -> (body, dict)`** — returns `(unchanged_text, {})` if there is no `---` header; otherwise returns `(body_without_header, parsed_dict)`; quoted YAML values are unquoted.
- **`reasons.api.import_beliefs(path, db_path)`** — parses a markdown belief file; returns `{"claims_imported": N, ...}`.

### Unused imports

`pytest`, `Justification`, and `import_into_network` are imported but never referenced in this file (no `pytest.raises`, no justification objects, no direct call to `import_into_network`). They are safe to remove or are placeholders for future tests.

---

## 3. Patterns

- **Fixture-based isolation**: every integration test takes `tmp_path` and creates a fresh `.db` and `.json`/`.md` file, so no shared state.
- **Round-trip assertions**: rather than asserting on implementation details, tests export → re-import → re-export and assert on the observable contract (`project_name` survives, `claims_imported == 2`).
- **Backward-compatibility tests as first-class citizens**: `test_import_without_meta_is_safe` and `test_frontmatter_ignored_gracefully` deliberately construct *legacy* payloads; this encodes the "never break old files" policy.
- **Direct DB inspection** in `TestSqliteMeta` bypasses the API to verify the storage layer actually persists keys — it treats `network_meta` as a stable table name with `key`/`value` columns.
- **File-name-derived defaults**: `init_db` infers `project_name` from `Path(db_path).stem`, which is why the assertions compare against `"test"` and `"myproject"`.

---

## 4. Dependencies

**Imports (this file → production):**
- `reasons.api` — facade for init/add/export/import
- `reasons.metadata` — `build_meta`, `SCHEMA_VERSION`
- `reasons.network.Network`
- `reasons.export_markdown.export_markdown`
- `reasons.import_beliefs.strip_frontmatter` (and unused `import_into_network`)
- standard lib: `json`, `pathlib.Path`, `sqlite3`

**Imported by:** nothing — it is a leaf test module. It is collected by pytest via `tests/__init__.py` + `conftest.py`.

**Coupled to (implicit):**
- The SQLite schema of `network_meta` (table name, `key`/`value` columns)
- The exact string `"ftl-reasons/"` in the generator field
- The `SCHEMA_VERSION == "1.0"` literal used in the markdown assertion (`'schema_version: "1.0"'`)

Changing any of those production literals requires updating this file.

---

## 5. Flow

Typical per-test flow:

```
tmp_path ──► api.init_db(db_path, [project_name])
             │
             ├─ writes network_meta rows: schema_version, project_name,
             │  created_at, updated_at (project_name defaults to Path(db).stem)
             │
             ▼
        api.add_node(...)  ──► persists nodes; preserves created_at
             │
             ▼
        api.export_network(db_path) ──► dict with nodes + meta
             │
             ├─(JSON path)  Path.write_text(json.dumps(data))
             │              api.init_db(dst); api.import_json(file, dst)
             │              api.export_network(dst)["meta"]["project_name"]
             │              == original project_name
             │
             └─(MD path)    api.export_markdown(db_path)  ──►  "---\n...frontmatter...\n---\n# ..."
                            Path.write_text(md)
                            api.import_beliefs(md_file, dst) ──► claims_imported == 2
```

`strip_frontmatter` sub-flow (unit):
- If text does **not** start with `---`, return `(text, {})`.
- Otherwise split on the closing `---` fence, parse `key: value` lines, unquote `"..."` values, and return `(body, dict)`.

---

## 6. Invariants Enforced by This Suite

1. **`SCHEMA_VERSION` is embargoed at `"1.0"`.** Changing it will break `test_basic_fields`, `test_export_includes_meta`, and `test_frontmatter_present` simultaneously — which is the intended tripwire.
2. **`created_at` is immutable** across mutations. `test_preserves_created_at` and `test_save_preserves_created_at` both fail if `add_node` rewrites it.
3. **`project_name` defaults to the DB filename stem** — not `""`, not `"default"`, not the process name.
4. **`build_meta()` with no args produces an empty-string `project_name`** (an in-memory default, distinct from the file-derived default).
5. **Legacy payloads must import without error.** Absence of `meta` (JSON) or frontmatter (markdown) is not an error condition.
6. **Markdown frontmatter is YAML-shaped and starts exactly at byte 0** — `md.startswith("---\n")`.
7. **Quoted scalar values in frontmatter are unquoted on parse** (`updated_at: "2026-..."` → bare string).
8. **`node_count` reflects the persisted node count at export time**, not a cached value (`test_export_includes_meta` adds one node and expects `1`).

---

## 7. Error Handling

The tests are deliberately **positive-path and compat-path** — they do not assert on raised exceptions. That is itself an error-handling contract:

- `test_import_without_meta_is_safe` and `test_frontmatter_ignored_gracefully` prove that **missing metadata is not an error** — the importers swallow it and proceed (they expect `result["nodes_imported"] == 1` / `result["claims_imported"] == 1` rather than a raised exception).
- `strip_frontmatter` returns `({}, body)` rather than raising when there’s no frontmatter.
- There is **no test for malformed frontmatter** (e.g., unclosed `---`, invalid YAML, missing colon). Behavior for those is unspecified by this suite — a gap to probe.
- There is **no test for `created_at` being corrupted** in the DB or for a mismatched `schema_version` on import (whether it is rejected, upgraded, or ignored).

If you extend the file, add negative tests here (malformed YAML, schema-version mismatch, missing `network_meta` rows) to lock down which failures are hard errors vs. silent repairs.

---

## Topics to Explore

- [function] `reasons/metadata.py:build_meta` — Constructor that computes `schema_version`, `generator`, `created_at`, `updated_at` defaults; you'll want to see exactly what `generator` string format is emitted and whether `created_at=None` triggers `now()`.
- [function] `reasons/api.py:init_db` — Determines how `project_name` defaults to `Path(db_path).stem` and which `network_meta` rows are written on creation.
- [function] `reasons/import_beliefs.py:strip_frontmatter` — Parser that must handle quoted values (and, per the tests, only the happy path); worth reviewing for YAML edge cases it silently mishandles.
- [file] `reasons/export_markdown.py` — How frontmatter is serialized (`schema_version: "1.0"` quoting, `---` fences); any formatting change will break `TestMarkdownExportMeta`.
- [file] `tests/test_metadata_extended.py` — Likely covers the negative / edge cases this file omits (malformed frontmatter, schema-version mismatch); read before adding tests here to avoid duplication.

---

## Beliefs

- `json-export-includes-meta` — `api.export_network(...)` returns a dict containing a `meta` key with `schema_version`, `node_count`, `project_name`, and `generator`.
- `init-db-defaults-project-name-to-db-stem` — `api.init_db(db_path)` sets `project_name` to `Path(db_path).stem` (e.g. `myproject.db` → `"myproject"`).
- `created-at-is-immutable-across-saves` — After `api.init_db`, subsequent `api.add_node` calls preserve the original `created_at` value in `network_meta`.
- `import-without-meta-is-backward-compatible` — `api.import_json` succeeds on payloads lacking a `meta` key, imporing nodes without error.
- `markdown-frontmatter-is-yaml-at-byte-zero` — `export_markdown(net)` output starts with `---\n` and YAML frontmatter; `strip_frontmatter` returns `(text, {})` when no such header exists.
- `schema-version-is-pinned-to-1-0` — `SCHEMA_VERSION == "1.0"` and both JSON and Markdown exports assert this literal, so changing it requires updating this suite.
