---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_metadata.py

# `tests/test_metadata.py`

## 1. Purpose

This file is the integration test suite for the **metadata layer** of `ftl-reasons`. Its job is to guarantee that a consistent, versioned metadata envelope (schema version, project name, node count, generator string, timestamps) is correctly attached to, preserved through, and retrievable from every storage format the project supports: **JSON export**, **Markdown export/import** (via YAML frontmatter), and **SQLite** (via the `network_meta` key-value table).

It doesn't test the metadata logic in isolation (that's `test_metadata_extended.py`'s likely domain). Instead, it exercises the metadata as it flows *through* the public API surface — `api.export_network`, `api.import_json`, `api.export_markdown`, `api.import_beliefs`, and `api.init_db` — confirming the contract holds end-to-end.

## 2. Key Components

| Class | Responsibility |
|---|---|
| `TestBuildMeta` | Unit-tests the `build_meta()` factory: verifies it stamps `schema_version`, `project_name`, `node_count`, `generator`, `created_at`, `updated_at`, and that a pre-supplied `created_at` is not clobbered. |
| `TestJsonExportMeta` | Confirms `api.export_network()` embeds a `"meta"` key in the JSON payload with the correct fields. |
| `TestJsonImportMeta` | Confirms `api.import_json()` propagates `meta.project_name` into the target DB, and that a JSON payload **without** a `meta` key still imports cleanly (backward compat). |
| `TestMarkdownExportMeta` | Asserts `export_markdown()` emits a YAML frontmatter block (`---\n…\n---`) with `schema_version`, `node_count`, `generator`, and optional `project_name`. |
| `TestStripFrontmatter` | Tests the `strip_frontmatter()` helper: no-op on plain text, correct key/value extraction on frontmatter (including quoted values), and that the returned body is clean. |
| `TestMarkdownRoundTrip` | The "golden path" test: export a 2-node network to `.md`, import it into a fresh DB, assert `claims_imported == 2`. Also verifies old files without frontmatter still work. |
| `TestSqliteMeta` | Pokes directly into SQLite via `sqlite3` to verify the `network_meta` table is populated at `init_db` time, that `created_at` survives subsequent writes, and that `project_name` can be overridden. |

**Constants under test:**
- `SCHEMA_VERSION` — imported from `reasons.metadata`; the tests assert it equals `"1.0"` (via the `build_meta` output and the JSON export).

## 3. Patterns

- **Format-agnostic contract testing.** Each `Test*Meta` class pins the *same* set of fields (`schema_version`, `project_name`, `node_count`, `generator`, `created_at`, `updated_at`) against a different serialisation target. This is a "contract over implementation" pattern: the test doesn't care *how* metadata is stored, only that it *appears correctly* in the output.

- **`tmp_path` fixtures for isolation.** Every test that touches disk uses pytest's built-in `tmp_path` fixture, creating a throwaway `.db` or `.json` file. No shared state between tests.

- **Round-trip testing.** `TestMarkdownRoundTrip` and `TestJsonImportMeta` both follow the write→read→assert pattern, which is the strongest guarantee that serialisation is lossless for the fields under test.

- **Backward-compatibility guards.** Two tests explicitly construct "old-format" inputs (JSON with no `meta` key, Markdown with no frontmatter) and assert they still import. This is a defensive pattern: it prevents a future refactor from silently breaking files written by older versions.

- **Direct DB inspection for invariants.** `TestSqliteMeta` bypasses the API and runs raw `SELECT key, value FROM network_meta` to verify storage-level invariants (e.g., `created_at` is immutable across writes). This catches bugs that API-level tests would mask.

- **Two package trees.** The repo contains both `reasons/` and `reasons_lib/` with identical module names. The tests import from `reasons`, suggesting `reasons` is the active package and `reasons_lib` is either a legacy copy or a build artifact (see `pyproject.toml` for the source-of-truth declaration).

## 4. Dependencies

**Imports (what this file needs):**

| Import | Role in tests |
|---|---|
| `reasons.api` | Public CRUD: `init_db`, `add_node`, `export_network`, `import_json`, `export_markdown`, `import_beliefs` |
| `reasons.metadata.build_meta` | The factory under test in `TestBuildMeta` |
| `reasons.metadata.SCHEMA_VERSION` | The expected constant value |
| `reasons.network.Network` | In-memory graph used by `TestMarkdownExportMeta` (no DB) |
| `reasons.export_markdown.export_markdown` | Direct markdown serialisation (avoids the DB layer) |
| `reasons.import_beliefs.strip_frontmatter` | Frontmatter parser under test |
| `sqlite3` (stdlib, local import) | Raw table inspection in `TestSqliteMeta` |
| `json`, `Path` (stdlib) | Writing fixture files to `tmp_path` |

**Imported by (what depends on this file):**

Nothing in the production code imports a test file. This file is consumed by the **test runner** (pytest / CI workflow in `.github/workflows`). It has no downstream dependents.

## 5. Flow

```
Test execution order (alphabetical by class, then by method):

TestBuildMeta
  ├── test_basic_fields          → build_meta() → assert 6 keys present
  ├── test_preserves_created_at  → build_meta(created_at=…) → assert not overwritten
  └── test_empty_project_name    → build_meta() → assert project_name == ""

TestJsonExportMeta              (each test: init_db → add_node → export_network → assert)
  ├── test_export_includes_meta
  ├── test_export_preserves_project_name
  └── test_export_default_project_name

TestJsonImportMeta
  ├── test_import_preserves_meta  → init_db(1) → add → export → write JSON
  │                                 → init_db(2) → import_json → export → assert name
  └── test_import_without_meta_is_safe  → hand-craft JSON w/o "meta" → import → assert ok

TestMarkdownExportMeta          (in-memory Network, no DB)
  ├── test_frontmatter_present
  └── test_frontmatter_with_project_name

TestStripFrontmatter            (pure string→tuple, no I/O)
  ├── test_no_frontmatter
  ├── test_with_frontmatter
  └── test_quoted_values

TestMarkdownRoundTrip
  ├── test_export_import_preserves_beliefs  → init → add 2 nodes → export md → write
  │                                           → init(2) → import_beliefs → assert 2 claims
  └── test_frontmatter_ignored_gracefully   → hand-write old md → import → assert 1 claim

TestSqliteMeta                 (raw sqlite3 for assertions)
  ├── test_init_sets_metadata   → init_db → SELECT * FROM network_meta → assert rows
  ├── test_save_preserves_created_at → init → SELECT created_at → add_node → SELECT again
  └── test_project_name_override → init_db(project_name=…) → SELECT → assert
```

Data transformations tested:
1. **In-memory Network → Markdown string** (with YAML frontmatter prepended)
2. **In-memory Network → JSON dict** (with `"meta"` key injected)
3. **JSON dict → SQLite rows** (meta key → `network_meta` table)
4. **Markdown string → In-memory Network** (frontmatter stripped, body parsed)
5. **SQLite → `network_meta` key-value pairs** (queried directly)

## 6. Invariants

These are the rules the tests **enforce** and that production code must uphold:

1. **`schema_version` is always `"1.0"`** in every output format. It is read from the `SCHEMA_VERSION` constant, never hardcoded in the test (except the one `assert meta["schema_version"] == "1.0"` in `test_basic_fields` which pins the current value).

2. **`created_at` is immutable after initial write.** `test_preserves_created_at` and `test_save_preserves_created_at` both assert that a subsequent `add_node` does not overwrite the original timestamp. This is a critical audit invariant.

3. **`project_name` defaults to the DB filename (minus extension).** `test_export_default_project_name` shows that `init_db("…/test.db")` yields `project_name == "test"`. The test in `test_init_sets_metadata` confirms `"myproject.db"` → `"myproject"`.

4. **`generator` always contains the substring `"ftl-reasons/"`.** This is checked in both JSON and Markdown exports. It's a provenance stamp.

5. **Missing metadata is tolerated on import.** Both the JSON (no `"meta"` key) and Markdown (no frontmatter) backward-compat tests assert that imports succeed and data is preserved. The system must never raise on absent metadata.

6. **`node_count` reflects the actual node count at export time.** Verified as `1` in single-node exports and `2` implied by the round-trip test.

## 7. Error Handling

This file is a test suite, so its "error handling" is about **what failures it detects**:

- **Assertion failures** are the sole error mechanism. Every test uses bare `assert` (no `pytest.raises`), meaning any contract violation — missing key, wrong type, mismatched value — produces a clear `AssertionError` with the diff visible in pytest output.

- **No exception-path tests here.** There is no test for "what happens when the JSON file is malformed" or "what happens when the SQLite table is missing." Those negative-path tests likely live in `test_metadata_extended.py` or `test_import_json.py`. This file is the **happy-path + backward-compat** suite.

- **Silent degradation is the design for missing metadata.** `test_import_without_meta_is_safe` and `test_frontmatter_ignored_gracefully` encode the rule: absent metadata → proceed with defaults, do **not** raise. If a future refactor adds a hard requirement for `meta`, these two tests will catch the regression.

- **Local `sqlite3` imports inside test methods.** Three tests in `TestSqliteMeta` do `import sqlite3` *inside* the method body rather than at module top level. This is a minor style inconsistency (it's already a stdlib module) and has no functional effect, but a maintainer might hoist it to the top-level imports for clarity.

- **No `finally` / cleanup blocks.** The tests rely on `tmp_path` (pytest cleans up automatically) and on SQLite's file-on-disk model (no connections to close beyond the explicit `conn.close()` calls in `TestSqliteMeta`). The `TestSqliteMeta` methods do explicitly close connections, which is correct; forgetting would leak file handles but wouldn't fail the test.

---

## Topics to Explore

- [file] `reasons/metadata.py` — The production module under test. Understanding `build_meta`, `SCHEMA_VERSION`, and the metadata dict shape is essential before modifying any of these tests.
- [file] `reasons/export_markdown.py` — Contains `export_markdown()` which prepends the YAML frontmatter block. The exact formatting (quoting, key order) is pinned by `TestMarkdownExportMeta`.
- [file] `reasons/import_beliefs.py` — Houses `strip_frontmatter()` and `import_into_network()`. The frontmatter parser's handling of quoted vs. unquoted values is a subtle edge case covered here but easy to break.
- [file] `tests/test_metadata_extended.py` — Likely covers edge cases and error paths not tested here (malformed frontmatter, schema version mismatch, concurrent writes). Read it to understand the full metadata test surface.
- [general] `reasons-vs-reasons_lib-duplication` — The repo has two near-identical package trees. Understanding which is canonical (check `pyproject.toml` build config) prevents editing the wrong copy when adding new metadata fields.

---

## Beliefs

- `created-at-is-immutable-after-init` — Once `init_db` stamps `created_at` in the `network_meta` table, no subsequent API call (`add_node`, `export`, `import`) overwrites it; only `updated_at` advances.
- `meta-absence-enables-backward-compat` — JSON files without a `"meta"` key and Markdown files without a YAML frontmatter block are valid inputs; the import path must succeed and produce a network with default metadata rather than raising.
- `project-name-defaults-to-db-stem` — When `init_db` is called without an explicit `project_name`, the value is derived from the database filename's stem (e.g., `myproject.db` → `"myproject"`), not hardcoded or left as `None`.
- `schema-version-gates-format-contract` — `SCHEMA_VERSION` (currently `"1.0"`) appears in every serialisation target (JSON `meta`, Markdown frontmatter, SQLite `network_meta` row) and is the single field a consumer checks to decide whether it can parse the payload.
- `round-trip-preserves-claim-count` — Exporting a network to Markdown and re-importing it into a fresh database yields the same `claims_imported` count as the number of nodes in the source network; no claims are silently dropped or duplicated by the frontmatter stripping step.
