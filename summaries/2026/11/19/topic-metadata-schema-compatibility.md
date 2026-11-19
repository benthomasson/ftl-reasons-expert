---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Check how `schema_version = "1.0"` is consumed by importers/exporters and whether migration logic exists elsewhere.

## Summary

`schema_version = "1.0"` is currently a **write-time constant and pass-through metadata field**, not a versioned compatibility boundary. Importers copy it into `network.meta` when present, but the visible code does not validate, branch on, reject, or migrate based on its value. Exporters ask `build_meta()` for metadata and emit the current `SCHEMA_VERSION`, which is `"1.0"`.

There is migration logic in the repo, but it is **not schema-version migration**. The `migrat*` matches are for converting retraction reasons into graph-native defeaters, plus a storage comment about table creation. No code checks `schema_version` to transform older/newer formats.

## How `schema_version` is produced and exported

- `reasons/metadata.py:7` defines the source of truth: `SCHEMA_VERSION = "1.0"`.
- `reasons/metadata.py:31-54` `build_meta()` always returns `"schema_version": SCHEMA_VERSION`. It does not accept an incoming schema version.
- `reasons/export_card.py:40-45` builds metadata with only `project_name`, `node_count`, and `created_at`, then `reasons/export_card.py:68` writes `schema_version` into YAML frontmatter.
- `reasons/export_markdown.py:19-27` does the same via `build_meta()`, and `reasons/export_markdown.py:25` emits `schema_version` into frontmatter.
- `reasons/storage.py:195` and `reasons/pg.py:151` persist `schema_version` using `SCHEMA_VERSION`; `reasons/storage.py:326` also defaults missing metadata to `SCHEMA_VERSION`.

So exporters do not preserve an arbitrary imported `schema_version`; they normalize output to the current constant through `build_meta()`.

## How importers consume `schema_version`

### Markdown import via `import_beliefs`

- `reasons/import_beliefs.py:17-44` `strip_frontmatter()` does a simple `key: value` parse of YAML frontmatter.
- `reasons/import_beliefs.py:180-185` `import_into_network()` copies these keys into `network.meta` when present:
  - `schema_version`
  - `project_name`
  - `created_at`
- There is **no comparison, validation, or branching** on the parsed `schema_version`. `parse_beliefs()` then parses the markdown body the same way regardless of version.

### Agent import via `import_agent`

In the visible portion of `reasons/import_agent.py`:

- `_normalize_markdown()` calls `parse_beliefs()` directly and does not visibly read frontmatter metadata.
- `_normalize_json()` reads `data["nodes"]` and `data["nogoods"]`, but does not read `data["meta"]["schema_version"]`.
- The provided `import_agent.py` excerpt stops at line 200, so metadata handling could exist later, but it is not visible in the normalization paths shown.

### Other import/export metadata paths

- `reasons/api.py:2127` and `reasons/pg.py:2250` / `reasons/pg.py:2398` loop over `("schema_version", "project_name", "created_at")`. The exact surrounding functions are not included, but these appear to be metadata copy paths rather than version-validation paths.
- Without the surrounding context, we cannot prove they never validate, but the observed lines themselves are just key copying.

## Does schema migration logic exist?

No schema-version migration logic was found.

The `migrat` search hits are unrelated to `schema_version`:

- `reasons/api.py:1549` `migrate_retract_to_defeaters()` migrates string-based retraction reasons into defeater nodes.
- `reasons/cli.py:285` `cmd_migrate_defeaters()` exposes that as a CLI command.
- `reasons/cli.py:3155` registers `migrate-defeaters`.
- `reasons/storage.py:136` mentions checking tables before `SCHEMA` creates them “for migration”, but that is database/table setup, not `schema_version`-driven data migration.

There is no observed code that does something equivalent to:

```python
if schema_version == "0.9":
    upgrade_to_1_0(...)
elif schema_version != SCHEMA_VERSION:
    raise UnsupportedVersion(...)
```

## Practical implication

If a file declares `schema_version: "2.0"` or `schema_version: "0.9"`, the markdown importer will copy that value into `network.meta` and parse the body using the current grammar anyway. If that network is later exported through `export_card()` or `export_markdown()`, `build_meta()` will overwrite the version with `"1.0"`.

The tests do not cover real migration behavior. `tests/test_metadata_extended.py:64-80` verifies round-trip survival only with the current `"1.0"` value, so it cannot distinguish “preserved” from “normalized to the only supported version.”

## What is missing from the observations

To fully close the audit, we would need the surrounding implementations around:

- `reasons/api.py:2127`
- `reasons/pg.py:2250`
- `reasons/pg.py:2398`
- `reasons/storage.py:326`
- the full `reasons/import_agent.py` file beyond the first 200 lines
- the JSON import/export functions where `data["meta"]` is handled

The visible evidence strongly suggests no schema-version migration or compatibility gate exists, but those contexts would confirm it.

## Topics to Explore

- [function] `reasons/import_beliefs.py:import_into_network` — Confirms that frontmatter `schema_version` is copied into `network.meta` without validation or branching.
- [file] `reasons/api.py` — Inspect the import/export JSON paths around line 2127 to see whether schema metadata is validated elsewhere.
- [function] `reasons/api.py:migrate_retract_to_defeaters` — The only substantive migration logic found; confirm it is unrelated to schema-version upgrades.
- [file] `reasons/import_agent.py` — Read the full file beyond the visible first 200 lines to verify whether agent imports ignore or preserve `schema_version`.
- [general] `schema-version-compatibility` — Search for version comparisons, rejection paths, or upgrade hooks; current observations show none.

## Beliefs

- `schema-version-is-constant` — `SCHEMA_VERSION` is defined as `"1.0"` in `reasons/metadata.py` and `build_meta()` always emits that value.
- `exporters-write-schema-version-from-build-meta` — `export_card()` and `export_markdown()` emit `meta["schema_version"]` from `build_meta()` in YAML frontmatter.
- `import-beliefs-copies-schema-version-without-validation` — `import_into_network()` copies frontmatter `schema_version` into `network.meta` but does not compare or validate it.
- `import-agent-normalizers-ignore-schema-version` — The visible `_normalize_markdown()` and `_normalize_json()` paths in `reasons/import_agent.py` do not read `schema_version`.
- `no-schema-version-migration-exists` — The `migrat` matches refer to retract-to-defeater migration and table migration comments, not schema-version data migration.
