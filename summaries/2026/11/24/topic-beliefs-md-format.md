---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Understand the external schema this output claims compatibility with, so changes here can be evaluated against a real spec.

## What external compatibility is claimed

There are two export paths with explicit compatibility claims:

1. **HuggingFace EEM card** — `reasons/export_card.py`
   - Module docstring: `"Export a reasons network as a HuggingFace-compatible EEM card (README.md)."` (`reasons/export_card.py:1`)
   - `export_card()` builds a Markdown `README.md` with YAML frontmatter plus body sections.
   - The frontmatter is the main schema surface. It includes:
     - `schema_version` from `build_meta(...)` (`reasons/export_card.py:68`)
     - `type: eem`
     - `project_name`, `domain`, `license`, `base_network`, `source_repos`
     - stats: `beliefs_total`, `beliefs_in`, `beliefs_out`, `premises`, `derived`, `nogoods`
     - `generator`
   - The body includes `# {title}`, `## Stats`, `## How to Use`, `## Files`, `## Quality`, `## Limitations`, and `## License`.

2. **beliefs.md-compatible Markdown** — `reasons/export_markdown.py`
   - Module docstring: `"Export a reasons network as beliefs.md-compatible markdown."` (`reasons/export_markdown.py:1`)
   - `reasons/api.py:2546` repeats: `"Export the network as beliefs.md-compatible markdown."`
   - Frontmatter includes `schema_version` (`reasons/export_markdown.py:25`), `project_name`, `updated_at`, `node_count`, and `generator`.
   - Body uses `# Belief Registry`, optional `## Repos`, `## Claims`, and `## Nogoods`.
   - Each claim is emitted as `### {node.id} [{status}] {btype}` with bullets such as `- Depends on:`, `- Unless:`, `- Defeated by:`, `- Stale reason:`, `- Superseded by:`.

A third, separate external contract exists in `reasons/mcp_server.py`: it exposes tools through `FastMCP` and `@mcp.tool()` decorators, so the external schema is the MCP tool schema generated from Python signatures and docstrings. That is not the export-card schema, but it is another place where changing signatures/docstrings changes an external contract.

## Where the real spec is missing

The observations do **not** include an actual EEM spec URL, a JSON Schema, an OpenAPI/Swagger document, or a `$schema` reference. The URL search found HuggingFace constants such as `HF_BASE = "https://huggingface.co"` (`reasons/hf.py:10`) and HuggingFace repo URL construction in `reasons/api.py:2398`, but no vendored EEM or beliefs.md schema.

The code also does not validate the exported string against a schema. `export_card()` simply assembles lines and returns `"\n".join(lines)`. So the compatibility is declarative: the docstring says “HuggingFace-compatible EEM card,” but the repository does not contain the external spec used to verify that claim.

The `schema_version` value is internal metadata. `reasons/metadata.py` defines/builds metadata including `schema_version` (`reasons/metadata.py:37`, `:49`), and import paths check keys like `schema_version`, `project_name`, and `created_at` (`reasons/import_beliefs.py:183`, `reasons/api.py:2127`). That is not necessarily the same as the external EEM spec version.

## How to evaluate changes against a real spec

- For `reasons/export_card.py`, fetch the real HuggingFace model-card metadata rules and the EEM card spec. Check required frontmatter fields, allowed types, whether `null` vs empty list is valid, how `type: eem` is interpreted, and whether custom fields like `beliefs_total` are allowed.
- For `reasons/export_markdown.py`, compare against the beliefs.md spec and the in-repo consumer/parser in `reasons/import_beliefs.py`. Ensure frontmatter keys and `## Claims` / `### {id} [{status}] {btype}` headings round-trip.
- For `reasons/mcp_server.py`, treat the FastMCP-generated tool JSON schema as the external contract. Changing a tool’s parameter names, type hints, defaults, or docstring can change what MCP clients see.
- Add contract tests/fixtures: golden `README.md` for `export_card`, golden `beliefs.md` for `export_markdown`, and schema snapshots for MCP tools.

## Topics to Explore

- [file] `reasons/export_card.py` — Implements the claimed HuggingFace EEM card output; inspect frontmatter fields and body structure before changing compatibility behavior.
- [file] `reasons/export_markdown.py` — Implements the claimed beliefs.md-compatible output; compare headings and bullet fields with the import/parser path.
- [function] `reasons/metadata.py:build_meta` — Defines the internal `schema_version` and metadata keys embedded into exports.
- [file] `reasons/import_beliefs.py` — Likely in-repo consumer of beliefs.md frontmatter/claims; useful as a de facto contract for round-tripping.
- [general] `huggingface-eem-card-spec` — The external spec is not vendored here; fetch and compare the real required schema before making compatibility changes.

## Beliefs

- `export-card-claims-huggingface-eem-compatibility` — `reasons/export_card.py` explicitly claims to generate a HuggingFace-compatible EEM card `README.md`.
- `export-card-frontmatter-uses-schema-version-and-type-eem` — `export_card()` emits YAML frontmatter containing `schema_version` from `build_meta` and `type: eem`.
- `export-markdown-claims-beliefs-md-compatibility` — `reasons/export_markdown.py` and `reasons/api.py:2546` claim to produce beliefs.md-compatible Markdown.
- `schema-version-is-internal-metadata` — `reasons/metadata.py` defines `schema_version` via `build_meta`, so it is internal metadata rather than necessarily the external EEM spec version.
- `no-external-schema-url-found-in-observed-matches` — The observed URL/schema search found HuggingFace and GitHub URLs but no `$schema`, OpenAPI/Swagger, or EEM spec URL.
