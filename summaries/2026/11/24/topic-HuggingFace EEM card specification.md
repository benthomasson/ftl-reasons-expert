---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Look up the external HF/EEM schema to confirm required vs optional fields.

## Answer

The observations do **not** include the external HF/EEM schema, so we cannot confirm from this codebase whether a given EEM field is required or optional according to the external Hugging Face / EEM-Hub spec. `find_hf_eem_docs` returned **0 matches**, and `reasons/hf.py` only implements transport: download, create repo, upload. It does not validate `network.json` against a schema.

What the code does establish:

- `reasons/hf.py:download_network` downloads a fixed path:
  `https://huggingface.co/{repo_id}/resolve/main/network.json`
  (see the URL construction in `download_network`, around `reasons/hf.py:68`). It returns the raw decoded JSON string and only handles HTTP 401/404. There is no field-level validation.
- Publishing is documented as uploading three files: `network.json`, `beliefs.md`, and `README.md` (EEM card) — see `reasons/api.py:2352`.
- The EEM card generator emits Hugging Face frontmatter with `type: eem` — see `reasons/export_card.py:69`, and tests assert this in `tests/test_export_card.py` (`test_type_is_eem`).
- `tests/test_import_hf.py:SAMPLE_NETWORK` is the clearest local example of the expected network shape:
  - top-level: `meta`, `nodes`, `nogoods`, `repos`
  - `meta`: `schema_version`, `project_name`
  - node: `text`, `truth_value`, `justifications`, `source`, `source_hash`, `date`, `metadata`
- However, this is a test fixture, not the external schema.
- The local importer is forgiving:
  - `reasons/import_agent.py:_normalize_json` uses `.get("nodes", {})`, `.get("nogoods", [])`, and defaults missing node fields such as `text`, `truth_value`, `justifications`, `source`, `source_hash`, `date`, and `metadata`.
  - `reasons/import_beliefs.py` frontmatter handling copies `schema_version`, `project_name`, and `created_at` only if present; the same key list is visible at `reasons/api.py:2127`.
- This means the **local importer treats many fields as optional**, but that does **not** prove the external EEM schema treats them as optional.

What is missing:

- The actual external HF/EEM schema or EEM-Hub specification.
- Any schema-validation code for `network.json`.
- The full contents of `reasons/export_card.py` and any canonical EEM-Hub example repository.
- A canonical `network.json` from EEM-Hub to compare against `SAMPLE_NETWORK`.

To complete the lookup, inspect the external EEM-Hub schema/docs or a canonical EEM-Hub repo, then compare those required fields against `tests/test_import_hf.py:SAMPLE_NETWORK`, `reasons/import_agent.py:_normalize_json`, and `reasons/export_card.py`.

## Topics to Explore

- [general] `EEM-Hub external schema` — The actual external spec needed to confirm which EEM fields are required vs optional.
- [function] `reasons/hf.py:download_network` — Shows the fixed `network.json` path and that no schema validation happens during download.
- [file] `reasons/export_card.py` — Generates the Hugging Face EEM card frontmatter; inspect all emitted fields and optionality.
- [file] `tests/test_import_hf.py` — Contains `SAMPLE_NETWORK`, the local canonical test fixture for import shape.
- [function] `reasons/import_agent.py:_normalize_json` — Shows which JSON fields the local importer defaults rather than requiring.

## Beliefs

- `external-hf-eem-schema-not-in-observations` — `find_hf_eem_docs` returned 0 matches, and no external schema definition appears in the provided observations.
- `download-network-fetches-fixed-path` — `reasons/hf.py:download_network` fetches `network.json` from `.../resolve/main/network.json` and returns raw decoded JSON without schema validation.
- `import-agent-json-normalizer-defaults-missing-fields` — `reasons/import_agent.py:_normalize_json` defaults missing top-level `nodes`/`nogoods` and missing node fields instead of raising.
- `import-hf-test-fixture-has-meta-nodes-nogoods-repos` — `tests/test_import_hf.py:SAMPLE_NETWORK` includes top-level `meta`, `nodes`, `nogoods`, and `repos`.
- `publish-uploads-three-files` — `reasons/api.py:2352` documents publish as uploading `network.json`, `beliefs.md`, and `README.md` (EEM card).
