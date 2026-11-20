---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: What HuggingFace's EEM card format prescribes, to understand why `type: eem`, `base_network`, and `source_repos` exist in the frontmatter.

The observations do **not** include HuggingFace’s official EEM card specification. What they do show is this repo’s implementation of an “HuggingFace-compatible EEM card” as a `README.md`-style document with YAML frontmatter.

`reasons/export_card.py` describes the output as a “HuggingFace-compatible EEM card (README.md)” (`reasons/export_card.py:1`), and the tests parse the frontmatter with `yaml.safe_load` between `---` delimiters (`tests/test_export_card.py:11-14`). So the concrete contract visible here is: **EEM metadata lives in machine-readable YAML frontmatter, and the Markdown body carries human-readable stats/instructions**.

## What the frontmatter prescribes in this codebase

`export_card()` accepts optional `base_network` and `source_repos` arguments (`reasons/export_card.py:34-35`) and always emits them in the frontmatter:

| Field | What the code does | Why it exists |
|---|---|---|
| `type: eem` | Hard-coded literal at `reasons/export_card.py:69`; asserted by `test_type_is_eem` (`tests/test_export_card.py:49-53`). | Acts as a card-type discriminator: this is an EEM card, not a generic HF model card. |
| `base_network` | If provided, writes `base_network: "<id>"`; otherwise writes `base_network: null` (`reasons/export_card.py:82-85`). Tests cover both branches (`tests/test_export_card.py:85-95`). | Records lineage: the parent EEM this network was derived from. Docstring says “Parent EEM this was derived from” (`reasons/export_card.py:43`). |
| `source_repos` | If provided, writes a YAML list; otherwise writes `source_repos: []` (`reasons/export_card.py:87-92`). Tests cover list serialization (`tests/test_export_card.py:97-101`). | Records provenance/attribution: source repository identifiers. Docstring says “Source repository identifiers” (`reasons/export_card.py:44`). |

The full frontmatter block also includes `schema_version`, `project_name`, `domain`, `license`, belief counts, counts of premises/derived beliefs/nogoods, and `generator` (`reasons/export_card.py:65-96`). `schema_version` comes from `metadata.SCHEMA_VERSION` via `build_meta` (`reasons/metadata.py`, where `SCHEMA_VERSION = "1.0"`).

These fields are passed through the API/CLI into the exporter: see `reasons/api.py:2345-2383` and `reasons/api.py:2612-2621`, plus `reasons/cli.py:1012-1085`. That confirms they are intended as first-class export metadata, not incidental strings.

So, in this implementation, the `type`, `base_network`, and `source_repos` frontmatter fields exist to make the card self-describing on HuggingFace: what kind of card it is, what parent EEM it derives from, and what source repositories contributed to it.

**Caveat:** whether HuggingFace formally prescribes these exact field names, or whether they are local EEM extensions, is not visible in the observations. The observations show the generator contract, not the external specification.

## Topics to Explore

- [file] `reasons/export_card.py` — Full EEM frontmatter and Markdown body generation.
- [function] `reasons/export_card.py:export_card` — Builds the card string, including optional lineage fields.
- [file] `reasons/import_beliefs.py` — Shows how frontmatter is stripped and mapped into `network.meta`.
- [file] `reasons/hf.py` — HuggingFace repo resolution, download, create, and upload mechanics.
- [general] `HuggingFace EEM card specification` — Look up the external HF/EEM schema to confirm required vs optional fields.

## Beliefs

- `eem-card-is-readme-with-yaml-frontmatter` — `export_card()` returns a README-style string with YAML frontmatter delimited by `---`.
- `eem-card-type-is-hardcoded-eem` — The generated frontmatter always sets `type` to `"eem"`.
- `base-network-records-parent-eem` — `base_network` is emitted as a quoted parent ID or `null` when absent.
- `source-repos-records-provenance` — `source_repos` is emitted as a YAML list or `[]` when absent.
- `eem-card-schema-version-comes-from-metadata` — `schema_version` is populated from `metadata.SCHEMA_VERSION` through `build_meta`.
