---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `source_type` fits into the broader node metadata model, and whether the allowlist is expected to grow or migrate.

`source_type` is an **optional per-node provenance/category tag**, not part of the network-level metadata model. The network/export metadata is built separately in `reasons/metadata.py:build_meta`, which returns `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, and `generator`. That `SCHEMA_VERSION = "1.0"` applies to the network export metadata, not to the node `source_type` vocabulary.

For nodes, `source_type` is handled in `reasons/api.py`:

- `SOURCE_TYPES` is hard-coded at `reasons/api.py:274` as `{"code", "document", "self-description", "derived"}`.
- `add_node` accepts `source_type: str = ""` at `reasons/api.py:290`.
- If `source_type` is non-empty, it must be in `SOURCE_TYPES`; otherwise `ValueError` is raised at `reasons/api.py:370-373`.
- A valid, non-empty value is stored as `metadata["source_type"]` at `reasons/api.py:381-382`.
- Empty values are treated as absent and are not stored; this is asserted by `tests/test_source_type.py:test_empty_type_not_stored`.

So within node metadata, `source_type` is a special-cased, validated key inside an otherwise loose metadata dict. It is projected back out in exports and displays, e.g. `reasons/api.py:3501`, `reasons/cli.py:534-535`, and `reasons/export_markdown.py:69-70`. It also round-trips through markdown: `reasons/import_beliefs.py:115` parses `- Source type:` and `reasons/import_beliefs.py:249-250` writes it into node metadata. Derived nodes are automatically tagged with `source_type="derived"` in `reasons/derive.py:663`.

There is also an adjacent **source-record** `source_type` path: `reasons/storage.py:83` defines a `source_type TEXT NOT NULL DEFAULT ''` column, and `add_source` accepts/inserts it at `reasons/api.py:1007-1024`. That path is distinct from `add_node`'s node-metadata validation. The only allowlist check observed is in `add_node`; `SOURCE_TYPES` is referenced only at `reasons/api.py:274`, `370`, and `373`.

### Is the allowlist expected to grow or migrate?

The observations do **not** show a documented roadmap, migration code, or forge metadata for `source_type`; `source_type_docs` and `source_type_forge_metadata` both have `total_count: 0`. The tests hard-code the four allowed values and assert that `"unknown"` raises. There is no database enum or check constraint observed for this vocabulary; storage uses `TEXT`. Also, `reasons/import_beliefs.py:249-250` writes `source_type` into metadata directly without an observed `SOURCE_TYPES` check, so imports can bypass the `add_node` allowlist.

That means the allowlist is an **API-level validation policy**, not a broad data-integrity invariant. If a new type is added, the minimal code change is to extend `SOURCE_TYPES` and update tests; because values are stored as text/metadata, adding a new value likely does not require a storage migration. If types are renamed or removed, or if legacy/imported data must be normalized, migration/backfill logic would be needed—but none is present in the observations. The `schema_version: "1.0"` in `reasons/metadata.py` is network metadata and is not shown to version the `source_type` allowlist.

## Topics to Explore

- [function] `reasons/api.py:add_node` — Central entry point that validates `source_type` against `SOURCE_TYPES` and writes it into node metadata.
- [function] `reasons/import_beliefs.py:import_into_network` — Writes `source_type` directly into node metadata, so it can bypass `add_node` validation.
- [file] `reasons/storage.py` — Defines the `source_type TEXT` column and persistence behavior; check whether any schema constraints or load/save validation exist.
- [function] `reasons/api.py:add_source` — Separate source-record path that accepts `source_type` without an observed allowlist check.
- [general] `source-type-migration` — Look for release notes, migrations, or issue context that indicate whether the four-value vocabulary is intended to grow or be normalized.

## Beliefs

- `source-type-allowlist-is-hard-coded` — `SOURCE_TYPES` is a four-value set in `reasons/api.py` and is the only observed allowlist for node `source_type`.
- `add-node-stores-source-type-in-node-metadata` — A non-empty valid `source_type` is written to `node.metadata["source_type"]`; empty values are omitted.
- `source-type-validation-is-not-universal` — `add_node` validates against `SOURCE_TYPES`, but `import_into_network` writes `source_type` metadata directly and `add_source` has no observed allowlist check.
- `derived-nodes-auto-set-source-type` — Derivation code sets `source_type="derived"` in `reasons/derive.py`, so `derived` is system-generated as well as user-selectable.
- `network-metadata-does-not-include-source-type` — `build_meta` in `reasons/metadata.py` builds network/export metadata only and does not define or validate node `source_type`.
