---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Explore the matching endpoint on the service side (Meta schema_version `"1.0"`, `/api/agents/{id}/export`, `/beliefs`) to confirm round-trip compatibility.

## What the observations actually show (and what they don't)

The task asks you to confirm round-trip compatibility against a **service-side** surface: a `Meta.schema_version == "1.0"` field, an `/api/agents/{id}/export` route, and a `/beliefs` route. The first thing to flag is that **none of those HTTP routes appear in the observations.** Both discovery searches returned empty:

- `find_export_and_beliefs_routes` — `pattern: "/api/agents|/beliefs|export"` → `total_count: 0`
- `find_schema_version_usage` — `pattern: "schema_version|class Meta|Meta\\("` → `total_count: 0`

Those zeros are themselves suspicious. `reasons/metadata.py` plainly contains the literal `"schema_version"` key (see below), so the search must have been scoped to a route manifest / handler directory that doesn't exist, or it simply missed the files. **Don't trust the zero-match as evidence of absence — trust the file reads.**

The confusion is that the file named `reasons/api.py` is **not** an HTTP API. Its module docstring (lines ~1–8 of `reasons/api.py`) says:

> "This module provides standalone functions that any Python caller can use (CLI, LangGraph tools, scripts) without dealing with Storage lifecycle or argparse. Each function opens the database, operates, saves, and closes. All functions return dicts suitable for JSON serialization."

So "the service side" in this codebase is a **library of standalone functions plus CLI commands**, not a FastAPI/Flask router. There is no route table to match `/api/agents/{id}/export` or `/beliefs` against — at least not in the evidence provided.

## What *is* confirmed about `schema_version: "1.0"`

The Meta contract is genuinely present and centralized in `reasons/metadata.py`:

- `SCHEMA_VERSION = "1.0"` (~line 11) is the single source of truth.
- `build_meta(project_name, node_count, created_at)` (starting ~line 22) returns a dict with exactly: `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, `generator` (where `generator` is `f"ftl-reasons/{version}"` via `_get_generator`).
- `export_card` in `reasons/export_card.py` calls `build_meta(...)` and embeds the value into YAML frontmatter as `schema_version: "{meta["schema_version"]}"` (in the `lines = [...]` block near the frontmatter assembly). The generated `README.md`/EEM card therefore declares `schema_version: "1.0"`.
- `reasons/api.py` imports `from .metadata import build_meta` (imports block near the top, ~line 21), so the same metadata builder is used by the library API surface.

That covers the **Meta side** of the question: there is a versioned schema, it is `"1.0"`, and it is emitted into the exported artifact.

## What is confirmed about the two "endpoint" contracts (as library functions)

Because there are no HTTP routes, the closest analogues are the export/import functions exercised by `tests/test_roundtrip.py`:

**`/beliefs` analogue → markdown export/import.**
- `export_markdown(network)` (in `reasons/export_markdown.py`, not included in the observations but used throughout `tests/test_roundtrip.py`).
- `import_into_network(network, beliefs_text, nogoods_text)` in `reasons/import_beliefs.py` (~line 165) parses that markdown back, and crucially **round-trips the frontmatter**: it copies `schema_version`, `project_name`, and `created_at` from the markdown frontmatter into `network.meta` (the `for key in ("schema_version", "project_name", "created_at"):` loop near the top of `import_into_network`).

Confirmed round-trip fields from `TestFullRoundTrip.test_all_fields_survive` (~line 130):
- `text`, `source`, `source_url`, `source_hash`, `date`
- `metadata["source_type"]` (code / document / derived / self-description)
- `metadata["accepted_pr"]`
- Justifications (type, antecedents, outlist)
- Status mapping: `STALE`/`OUT` → retracted nodes (`test_stale_node_round_trip`)
- `frontmatter` (`project_name`) survives (`test_frontmatter_round_trip`)
- `repos` block survives (`test_repos_preserved`)
- Double round-trip is stable (ignoring `updated_at`): `test_double_round_trip_stability`

**`/api/agents/{id}/export` analogue → the agent import path.**
`reasons/import_agent.py` normalizes **both** input formats — `_normalize_markdown` (~line 145) and `_normalize_json` (~line 180). The JSON shape it expects is therefore the de-facto "export" contract:

- Top-level `data["nodes"]`: dict of `node_id → { truth_value, text, source, source_url, source_hash, date, metadata, justifications }`
- Each justification: `{ type, antecedents, outlist, label }`
- Top-level `data["nogoods"]`: list of `{ id, nodes, discovered, resolution }` (`_normalize_nogoods_json` ~line 220)

`_normalize_json` also filters `outlist` entries to `node_ids` present in the payload (line ~185), so **any export endpoint must include every referenced outlist node in the same response** or those edges silently disappear on import.

## Where the round-trip is *not* confirmed

1. **No HTTP layer exists in the evidence.** There is no router, no request handler, no response envelope to verify that `/api/agents/{id}/export` produces the exact `nodes`/`nogoods` structure `_normalize_json` consumes. The contract above is inferred from the *consumer*, not the *producer*.
2. **No JSON exporter was read.** `export_card.py` produces a HuggingFace EEM card (YAML frontmatter + markdown), not `network.json`, even though its README table advertises `network.json`. The actual serializer that emits the `nodes`/`nogoods` JSON is not in the observations.
3. **`schema_version` is not validated on import.** `import_into_network` copies the frontmatter `schema_version` into `network.meta` but does not check it against `SCHEMA_VERSION`. A `"0.9"` payload would import silently. The `/beliefs` and agent-import paths would need an explicit version gate for real compatibility guarantees.
4. **Nogood markdown round-trip is documented as broken.** `tests/test_roundtrip.py::TestFullRoundTrip::test_nogoods_export_format` (~line 240) explicitly notes: *"parse_nogoods expects 'nogood-N: label' format but export writes 'nogood-NNN' without colon. Full nogood round-trip via markdown requires a format fix (pre-existing gap)."* `import_beliefs.parse_nogoods` (~line 128) matches `^### (nogood-\S+?):\s*(.*)` — the colon is required. So nogoods only round-trip reliably via the JSON path (`_normalize_nogoods_json`), not the markdown path.
5. **Agent-import namespacing is a transformation, not a pure round-trip.** `import_agent._import_claims` (~line 305) prefixes every node with `agent_name:`, injects `agent:inactive` into every outlist (`_build_justifications` ~line 265), and adds `agent:active` / `agent:inactive` infra nodes (`_ensure_agent_nodes` ~line 50). So a `/api/agents/{id}/export` response, when re-imported, will **not** be byte-identical — it will be namespaced and gain kill-switch outlist edges. Any "round-trip compatibility" check must account for this.

## Bottom line

From these observations you can confirm the **schema/Meta half** (`schema_version: "1.0"` centralized in `reasons/metadata.py`, emitted via `build_meta` into `export_card.py` and available to `reasons/api.py`) and the **beliefs round-trip half** for markdown/JSON at the field level (source_url, accepted_pr, source_type, source_hash, date, project_name, repos, LOG/OUT status) via `tests/test_roundtrip.py`. You **cannot** confirm the `/api/agents/{id}/export` or `/beliefs` *service-side* contracts because no HTTP handler was captured — and the pattern searches that should have found them returned 0, which conflicts with the known contents of `metadata.py`.

## Topics to Explore

- [file] `reasons/export_markdown.py` — The producer of the `/beliefs`-equivalent payload; verify its frontmatter and per-claim field emission matches `import_beliefs.parse_beliefs`
- [file] `reasons/import_agent.py` — Contains `_normalize_json`, the authoritative consumer contract for `/api/agents/{id}/export`; confirm the JSON exporter emits `nodes`/`nogoods` in this shape
- [general] `agent-export-endpoint-source` — Search the repo for a web/router module (FastAPI/Flask/ASGI) that actually registers `/api/agents/{id}/export` and `/beliefs`; the current searches found none
- [function] `reasons/metadata.py:build_meta` — Confirm every export surface (markdown, card, JSON, DB) funnels through this one function so `schema_version` cannot drift
- [general] `schema-version-validation` — Check whether any importer rejects a payload whose `schema_version != "1.0"`; `import_into_network` currently just copies it into `network.meta`

## Beliefs

- `schema-version-single-source` — `SCHEMA_VERSION = "1.0"` in `reasons/metadata.py` is the sole definition of the schema version, consumed through `build_meta` by `export_card.py` and `reasons/api.py`.
- `export-card-emits-schema-version-frontmatter` — `export_card` writes `schema_version: "1.0"` into the YAML frontmatter of the generated EEM card.
- `beliefs-import-preserves-frontmatter-meta` — `import_into_network` copies `schema_version`, `project_name`, and `created_at` from markdown frontmatter into `network.meta`.
- `nogood-markdown-roundtrip-is-broken` — `parse_nogoods` requires a `nogood-N:` colon header while the markdown exporter emits `nogood-NNN`, so nogoods only round-trip via the JSON path (`test_nogoods_export_format` documents the gap).
- `agent-json-import-requires-nodes-and-nogoods` — `_normalize_json` in `reasons/import_agent.py` expects a top-level `nodes` dict keyed by id with `truth_value` and `justifications`, plus a top-level `nogoods` list.
