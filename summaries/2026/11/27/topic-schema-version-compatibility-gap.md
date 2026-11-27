---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Decide whether to add a real version gate (compare `meta["schema_version"]` to `SCHEMA_VERSION` on open/import) or document that format drift is unsupported by design.

## Decision

**Document that cross-version format drift is unsupported by design for now; do not add a hard equality gate yet.** The current code treats `schema_version` as provenance/metadata, not as an enforced compatibility contract. If you want some protection without breaking existing import behavior, add a non-blocking warning on mismatch, not a rejection.

## Why this matches the current code

- `SCHEMA_VERSION` is defined as `"1.0"` in `reasons/metadata.py:11`, and `build_meta()` writes it into exported metadata at `reasons/metadata.py:49`.
- Export/storage paths write the current version:
  - markdown frontmatter: `reasons/export_markdown.py:25`
  - JSON card export: `reasons/export_card.py:68`
  - SQLite metadata: `reasons/storage.py:195`
  - PostgreSQL metadata: `reasons/pg.py:151`
- Import paths copy `schema_version` but do **not** compare it to `SCHEMA_VERSION`:
  - `reasons/import_beliefs.py:183` copies frontmatter keys including `schema_version` into `network.meta`.
  - `reasons/api.py:2127` and `reasons/pg.py:2250` / `reasons/pg.py:2398` have similar metadata-key loops.
  - `reasons/storage.py:326` uses `meta.setdefault("schema_version", SCHEMA_VERSION)`, which defaults missing values rather than rejecting them.
- Existing tests intentionally preserve backward compatibility:
  - `tests/test_metadata.py` has `test_import_without_meta_is_safe`.
  - `tests/test_metadata.py` has `test_frontmatter_ignored_gracefully`.
  - `tests/test_metadata_extended.py:80` expects round-trip export/import to preserve `schema_version`.
- The docs search found no schema-version compatibility or drift documentation. The only related hits are export documentation and a CLI `--version` mention.

## Why not a hard gate right now

A real gate needs a compatibility policy that does not currently exist:

- Should comparison be exact (`"1.0"` only) or major-version compatible?
- Should missing `schema_version` be allowed? Current tests say yes for old JSON/markdown.
- What should happen on mismatch: warning, error, or best-effort import?
- Which entrypoints must call the gate? At minimum `api.import_json`, `import_into_network`, SQLite load/open around `reasons/storage.py:326`, and PostgreSQL import paths around `reasons/pg.py:2250` / `reasons/pg.py:2398`.
- If you gate after copying metadata, a mismatched version may already have overwritten `network.meta["schema_version"]`. The check must happen before merge.

Those are reasonable to add later, but today they would be a behavior change not covered by tests.

## What to document instead

Document that:

- `SCHEMA_VERSION` is the only supported format version.
- `schema_version` is emitted for provenance/diagnostics.
- Importers may accept files with missing or foreign `schema_version` values, but cross-version correctness is not guaranteed.
- There is no migration path for format drift; users should keep backups before importing files from another version.

If/when a breaking format change is planned, add a small helper such as `metadata.check_schema_version(meta)` and call it from every import/open path before metadata is merged.

## Missing observations

The snippets strongly suggest there is no validation at `reasons/api.py:2127`, `reasons/storage.py:326`, or `reasons/pg.py:2250` / `reasons/pg.py:2398`, but the full bodies of those import/load functions are not included. Those should be read before implementing any gate.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full import/open path around line 2127 to confirm where JSON metadata is merged and whether any schema validation exists elsewhere.
- [file] `reasons/storage.py` — Read the SQLite load/save functions around line 326 to see how missing and mismatched `schema_version` values are handled.
- [file] `reasons/pg.py` — Check PostgreSQL import/load paths around lines 2250 and 2398 for metadata copying and potential validation points.
- [file] `tests/test_metadata_extended.py` — Review round-trip and schema-version tests to understand the current compatibility promises.
- [general] `schema-version-policy` — Decide exact-vs-major comparison, missing-version behavior, warning-vs-error semantics, and migration expectations before adding a gate.

## Beliefs

- `schema-version-constant-is-1-0` — `SCHEMA_VERSION` is `"1.0"` in `reasons/metadata.py:11`, and tests assert that value.
- `exports-emit-current-schema-version` — Metadata writers use `SCHEMA_VERSION` for `schema_version` in `reasons/metadata.py:49`, `reasons/export_markdown.py:25`, `reasons/storage.py:195`, and `reasons/pg.py:151`.
- `imports-copy-schema-version-without-comparison` — `import_into_network` copies frontmatter `schema_version` into `network.meta` without comparing it to `SCHEMA_VERSION` (`reasons/import_beliefs.py:183`).
- `missing-schema-version-imports-are-supported` — Tests require JSON without `meta` and markdown without frontmatter to import successfully (`tests/test_metadata.py`).
- `schema-drift-is-undocumented` — The docs search found no schema-version drift or compatibility documentation; matches were only export docs and CLI `--version`.
