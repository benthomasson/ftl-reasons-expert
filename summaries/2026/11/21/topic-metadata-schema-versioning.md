---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Explore how `schema_version == "1.0"` is used for compatibility and whether future migrations are planned.

# How `schema_version == "1.0"` Is Used for Compatibility

## Single source of truth

The version literal lives in exactly one place:

- `reasons/metadata.py:13` — `SCHEMA_VERSION = "1.0"`

Every other site imports and reuses this constant rather than hard-coding `"1.0"` as a string. That means bumping the version is a one-line change pending the tests below.

## Where the value is written (producers)

`schema_version` is attached to every serialization format via `build_meta()`:

- `reasons/metadata.py:49` — `"schema_version": SCHEMA_VERSION,` inside the dict assembled by `build_meta()`.
- `reasons/export_markdown.py:25` — renders it into markdown YAML frontmatter as `schema_version: "..."`.
- `reasons/export_card.py:68` — renders it into the "card" frontmatter.
- `reasons/storage.py:195` — on new SQLite DBs, inserts it into the `network_meta` key/value table (`CREATE TABLE IF NOT EXISTS network_meta (key TEXT PRIMARY KEY, value TEXT NOT NULL)` at storage.py lines ~63–66).
- `reasons/pg.py:151` — same insertion into the PostgreSQL metadata store.

So every export and every newly initialized DB carries `schema_version = "1.0"`.

## Where the value is read (consumers)

The import paths pull it back out as one of a small set of metadata keys:

- `reasons/import_beliefs.py:183` — `for key in ("schema_version", "project_name", "created_at"):`
- `reasons/api.py:2127` — same tuple (JSON import path).
- `reasons/pg.py:2250` and `reasons/pg.py:2398` — same tuple in PostgreSQL import paths.
- `reasons/storage.py:326` — `meta.setdefault("schema_version", SCHEMA_VERSION)` — a defaulting step on load.

Critically, **none of these read sites compare, gate, or branch on the value**. There is no `if meta["schema_version"] != SCHEMA_VERSION:` in the observed code, no `packaging.version` comparison, no rejection of files that declare an unknown schema version. The string is carried through as opaque metadata.

## Compatibility behavior in practice

Two design choices stand in for version-based compatibility:

1. **Graceful absence.** Missing metadata is tolerated, not rejected.
   - `storage.py:326` calls `meta.setdefault("schema_version", SCHEMA_VERSION)`, so a store without the key inherits the current version.
   - `tests/test_metadata.py::TestJsonImportMeta::test_import_without_meta_is_safe` documents that JSON without a `meta` key imports fine.
   - `tests/test_metadata.py::TestMarkdownRoundTrip::test_frontmatter_ignored_gracefully` documents that markdown with no frontmatter imports fine.

2. **Tests pin the value, not behavior.** `tests/test_metadata.py:19` and `tests/test_metadata_extended.py::test_schema_version_constant` assert `SCHEMA_VERSION == "1.0"`. These tests would fail on a bump — which is the actual enforcement point today (a "who bumped the version" guard rather than a "can I load older data" guard).

The net effect: `"1.0"` is a **forward-compatibility marker** that downstream tooling can read and inspect, but `ftl-reasons` itself does not act on it. Old data loads fine regardless of what the marker says.

## Where migration actually happens

The only real migration machinery in the observed code is **column-level, schema-agnostic, and unrelated to `schema_version`**:

- `reasons/storage.py:_init_schema` (line ~136 comment: `# Check which tables exist before SCHEMA creates them (for migration)`) — idempotent `ALTER TABLE ... ADD COLUMN` checks for `source_url`, `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`, `supporting_justification`, `text_hash`, and `content_hash`. It also backfills `node_tags` and `node_sources` from legacy `metadata_json`/`source` columns.
- These run on every `Storage(...)` initialization and are safe to re-run. They never consult `schema_version`.

The `find_migration_references` hits (`migrate_retract_to_defeaters` in `api.py:1549`, `migrate-defeaters` in `cli.py:285`) are **belief-model** migrations (converting string retract reasons into graph-native defeaters), not schema-version migrations.

## Is a future migration planned?

The observations do not show any of:

- docs (`find_schema_version_in_docs` → 0 matches)
- a `CHANGELOG.md` entry mentioning `schema_version` bumps or planned changes (the changelog stops at 0.18.0 and never touches metadata versioning)
- a comparison helper, `needs_upgrade()`, or `SCHEMA_VERSION` arithmetic anywhere
- a constant like `SCHEMA_VERSION_NEXT` or a `MIN_SUPPORTED_SCHEMA`

So the honest answer: **nothing in the codebase or docs signals a planned schema migration.** The version string appears to be intentional forward-compat scaffolding — it exists so that a future release *could* branch on it, and so the existing column-level migrations can keep working without it. If a v2.0 is ever introduced, the code would almost certainly need to add a comparison at the import sites (`import_beliefs.py:183`, `api.py:2127`, `pg.py:2250/2398`) plus a bump of the literal in `metadata.py:13`, and update the `test_schema_version_constant` test.

## What's missing from these observations

- The full bodies of the `for key in (...)` loops at `import_beliefs.py:183`, `api.py:2127`, and `pg.py:2250/2398` — we can see *which* keys are touched but not whether they are copied, filtered, or discarded.
- Whether any consumer outside this repo reads `schema_version` (the docs directory has zero hits, which is a signal but not proof).
- Any `pyproject.toml` package-version logic that might implicitly correlate with schema version.

## Topics to Explore

- [file] `reasons/import_beliefs.py` — Read the full `for key in ("schema_version", "project_name", "created_at"):` loop at line 183 to see whether the value is just copied into `net.meta` or validated.
- [function] `reasons/storage.py:Storage._init_schema` — The real, in-use migration mechanism (idempotent column additions); confirms schema evolution is handled structurally rather than by version number.
- [file] `tests/test_metadata_extended.py` — Contains `test_schema_version_constant` and round-trip tests; these are the enforcement points that would need updating for any future bump.
- [general] `schema-version-consumers` — Grep across the repo for `SCHEMA_VERSION` uses to confirm no comparison/lookup logic exists beyond the constant defined in `metadata.py`.
- [file] `CHANGELOG.md` — Check for any future entries referencing metadata or schema changes; currently none exist, which is the best available signal on long-term plans.

## Beliefs

- `schema-version-is-single-sourced` — The `"1.0"` schema version literal is defined only at `reasons/metadata.py:13` as `SCHEMA_VERSION`; every other site imports it.
- `build-meta-produces-schema-version` — `build_meta()` in `reasons/metadata.py` stamps `schema_version` into every metadata dict it returns, so all export formats (JSON, markdown, card) and both storage backends inherit it.
- `schema-version-is-written-not-read` — No observed code compares or branches on `schema_version`; import sites only copy it alongside `project_name` and `created_at`.
- `imports-tolerate-missing-schema-version` — Store loading uses `meta.setdefault("schema_version", SCHEMA_VERSION)` (`storage.py:326`), and JSON/markdown imports succeed when `meta`/frontmatter is absent, so the field is not a hard requirement.
- `migrations-are-column-additions-not-version-based` — `Storage._init_schema` (`storage.py:~136`) migrates by running idempotent `ALTER TABLE ... ADD COLUMN` checks, independent of `schema_version`.
