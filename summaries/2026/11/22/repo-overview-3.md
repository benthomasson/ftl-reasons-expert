---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Repo Overview

# Repository Overview

> **Note on evidence base:** the tree provided lists only the `tests/` directory. The application modules themselves live outside this listing. The architecture below is inferred from the test suite — which is unusually expressive here — and from the domain vocabulary it uses (`justification`, `defeater`, `outlist`, `nogood`, `dialectical`, `stale`, `dependents`). Where I make an architectural claim, I cite the specific test file that signals it.

## 1. Purpose

This is a **belief / knowledge management engine built around a Truth Maintenance System (TMS)** with defeasible reasoning and LLM-assisted acquisition. It manages a graph of beliefs where each belief is supported by *justifications* (premises → conclusion), can be attacked by *defeaters*, and has an IN/OUT status that must be recomputed whenever the graph changes.

The classic TMS vocabulary is unmistakable:
- `test_outlist.py` — TMS justifications carry an **outlist** (the premises required to be OUT).
- `test_hide_out.py` — hiding OUT-status beliefs.
- `test_nogood_id.py` — named *nogoods*, the failed-assumption constraints from assumption-based TMS/ATMS.
- `test_dialectical.py`, `test_classify_defeaters.py`, `test_defeat_justification.py`, `test_defeat_with_scope.py` — argumentation-theoretic defeat (rebut / undercut / undermine, with scope).
- `test_backtracking.py` — status recomputation / undo on belief retraction.

The problem it solves: **maintaining a consistent, justified, and repair-able body of knowledge when beliefs are added, attacked, contradicted, or go stale** — and exposing that knowledge store to humans (CLI/Markdown) and to LLMs (via MCP tools).

## 2. Architecture

Layered, service-oriented, with an event bus:

```
                 ┌──────────────────────────────────────────┐
  Human ─────────┤  CLI (test_cli)   HTTP API (test_api)    │
  LLM client ────┤  MCP server (test_mcp_server, ask_mcp)   │
                 └───────────────┬──────────────────────────┘
                                 │
                 ┌───────────────▼──────────────┐
                 │   Mind service (mind_service) │  ← orchestration / use-cases
                 │   ask / research / review /    │
                 │   summarize / verify / propose │
                 └──┬───────────┬───────────┬─────┘
                    │           │           │
       ┌────────────▼──┐ ┌──────▼──────┐ ┌──▼──────────────┐
       │ Core reasoning │ │  Ingestion  │ │  Presentation   │
       │ justifications │ │ import_*    │ │ export_card      │
       │ defeaters      │ │ hf / json   │ │ export_markdown  │
       │ contradictions │ │ agent / sync│ │ build_wiki       │
       │ repair / derive│ │ pin_sources │ │ report_belief    │
       │ dialectical    │ │             │ │ report_gated     │
       └────────┬───────┘ └──────┬──────┘ └──────────────────┘
                │                │
       ┌────────▼────────────────▼──────────┐
       │  Storage layer (storage, pg,       │  ← Postgres-backed
       │  pg_dispatch, metadata, merkle)    │     + pubsub bus
       └────────────────────────────────────┘
```

Design patterns:
- **Repository pattern** over Postgres (`test_storage.py`, `test_pg.py`) with the storage interface mocked out for the unit-level tests.
- **Event/publish-subscribe** for asynchrony (`test_pubsub.py`) — useful when recomputing IN/OUT status across many dependents.
- **Content-addressing with Merkle trees** (`test_merkle.py`) — dedupe and integrity verification of belief content.
- **Strategy/dispatch** (`test_pg_dispatch.py`) — routing queries to the correct SQL shape.

## 3. Key Components

Grouping the test files by responsibility tells you what the modules must be:

| Concern | Test evidence | Responsibility |
|---|---|---|
| Justifications | `test_add_justification.py`, `test_remove_justification.py`, `test_defeat_justification.py`, `test_outlist.py`, `test_nogood_id.py` | CRUD on justification records and their IN/OUT lists; named nogoods |
| Defeasible reasoning | `test_dialectical.py`, `test_classify_defeaters.py`, `test_defeat_with_scope.py`, `test_backtracking.py` | Defeat classification, scoped defeat, status propagation, backtracking |
| Consistency | `test_contradictions.py`, `test_deduplication.py`, `test_dependents_integrity.py`, `test_dangling_dependents.py` | Detect contradictions and orphaned dependents; repair |
| Repair | `test_repair.py`, `test_repair_premises.py`, `test_review_premises.py` | Propose fixes for unsupported premises |
| Derivation | `test_derive.py`, `test_derive_budget.py` | Forward derivation from premises (with a **token/compute budget** guard) |
| Staleness | `test_check_stale.py`, `test_check_stale_issue25.py` | Flag beliefs whose source has changed |
| Ingestion | `test_import_json.py`, `test_import_hf.py`, `test_import_beliefs.py`, `test_import_agent.py`, `test_sync_agent.py`, `test_pin_sources.py` | Normalize external knowledge into beliefs, pinned to a source |
| Storage | `test_storage.py`, `test_pg.py`, `test_pg_dispatch.py`, `test_metadata.py`, `test_merkle.py` | Persistence, metadata, content hashing |
| Query / ask | `test_ask.py`, `test_any_mode.py`, `test_llm.py` | Natural-language query over the belief graph, LLM-assisted answers |
| Research pipeline | `test_research.py`, `test_review.py`, `test_review_justifications.py`, `test_summarize.py` | Multi-step research → review/verify → summarize loop |
| Organization | `test_cluster.py`, `test_topics.py`, `test_network.py`, `test_network_cache.py` | Graph clustering, topic modeling, cached network views |
| Presentation | `test_export_markdown.py`, `test_export_card.py`, `test_build_wiki.py`, `test_report_belief.py`, `test_report_gated.py` | Render the belief graph as markdown, cards, wiki |
| Interfaces | `test_cli.py`, `test_api.py`, `test_mcp_server.py`, `test_ask_mcp.py` | CLI, HTTP, and MCP (Model Context Protocol) entrypoints |
| Service | `test_mind_service.py`, `test_propose_update.py` | The orchestration façade — likely the only layer the CLI/API touch |
| Cross-cutting | `test_access_tags.py`, `test_source_type.py`, `test_metadata_extended.py`, `test_compact.py` | Access control, source taxonomy, metadata, compaction |

## 4. Data Flow

The lifecycle of a belief:

1. **Ingest** — `import_json` / `import_hf` / `import_agent` normalizes a record. It is **pinned** to a source (`test_pin_sources.py`) and tagged with a source type (`test_source_type.py`) and access tags (`test_access_tags.py`).
2. **Deduplicate & hash** — content is Merkle-hashed (`test_merkle.py`); duplicates collide and are skipped (`test_deduplication.py`).
3. **Store with justification** — a justification (premises, `outlist`, optional nogood ID) is written atomically (`test_add_justification.py`, `test_nogood_id.py`).
4. **Propagate status** — adding a belief triggers a backtracking pass over *dependents* (`test_backtracking.py`, `test_dependents_integrity.py`). Contradictions are surfaced (`test_contradictions.py`); defeaters are classified and applied (`test_classify_defeaters.py`, `test_defeat_with_scope.py`).
5. **Repair / propose update** — the `review` and `repair` pipelines call the LLM to propose fixes (`test_propose_update.py`, `test_repair_premises.py`); a human or gate approves (`test_report_gated.py`).
6. **Query** — `ask` routes to the mind service, which may call the LLM with graph context (`test_llm.py`, `test_ask.py`); `test_any_mode.py` suggests a quantifier/mode switch (any-vs-all matching).
7. **Export / report** — `export_markdown`, `export_card`, or `build_wiki` renders the graph. `check_stale` can be run first to warn the reader (`test_check_stale.py`).

Outputs: Markdown, "cards", a static wiki, JSON (via `test_roundtrip.py`), and live MCP tool responses.

## 5. Dependencies

Inferred from test names and typical ecosystems — the `tests/` dir itself requires:

- **pytest + `conftest.py`** — with shared fixtures (likely temp Postgres or storage mock, mocked LLM, temp dirs).
- **A Postgres driver** (psycopg/asyncpg) — `test_pg.py`, `test_pg_dispatch.py` assume a real schema.
- **An LLM client** — `test_llm.py`, `test_import_agent.py`; likely Anthropic SDK given MCP.
- **`mcp` (Model Context Protocol)** — `test_mcp_server.py`, `test_ask_mcp.py`.
- **`huggingface_hub`** — `test_import_hf.py`, `test_hf_default_org.py`; imports datasets, defaults to a configured org.
- **A web framework** for the HTTP API (`test_api.py`) — FastAPI/Starlette is the common choice.
- **A CLI framework** (`test_cli.py`) — click or typer.
- **A pub/sub backend** (`test_pubsub.py`) — likely in-process async or Redis.

## 6. Entry Points

- **CLI** — `test_cli.py`; the primary human interface (import, ask, check-stale, export, review).
- **HTTP API** — `test_api.py`; programmatic access.
- **MCP server** — `test_mcp_server.py`, `test_ask_mcp.py`; exposes the belief graph as MCP *tools* and *resources* so an LLM client (e.g., Claude Desktop) can query and mutate the store.
- **Mind service** — `test_mind_service.py`; the internal façade. Almost certainly every other entry point talks to this and not to storage directly.

## 7. Configuration

Likely environment-driven, given the otherwise-Python project:

- Database URL / connection pool settings (`test_pg.py`).
- HuggingFace default org — `test_hf_default_org.py` implies a configurable default used when `import_hf` is called without an explicit org.
- LLM provider key + model (in `test_llm.py` and `test_derive_budget.py`).
- A **derivation budget** (tokens / nodes / wall-clock) — `test_derive_budget.py` implies this is configurable and enforced, not hard-coded.
- Access-tag policy (`test_access_tags.py`).

## 8. Module Boundaries

From the layer diagram, the intended import rules are:

- **Core reasoning does not import services** — the justifier/defeater engine must be usable without the mind service or MCP server (this is what makes `test_dialectical.py` fast and deterministic).
- **Storage is behind an interface** — `test_storage.py` vs. `test_pg.py` only makes sense if there is a storage protocol implemented by Postgres.
- **Presentation never writes** — exporters (`export_markdown`, `export_card`, `build_wiki`) are pure renderers.
- **Interfaces are thin** — the CLI / API / MCP servers only translate; all logic lives in the mind service. `test_ask_mcp.py` separating from `test_ask.py` confirms the split.
- **Ingestion depends on storage, never on presentation.**

## 9. Extension Points

- **New source importers** — add a sibling to `import_json` / `import_hf` / `import_agent` (e.g., `import_notion.py`), register the source type in `test_source_type.py`'s taxonomy.
- **New defeat classifiers** — extend the classification in `classify_defeaters` (a new defeater kind and its propagation rule).
- **New repair strategies** — add a strategy alongside `repair` and `repair_premises`; exercise it via `test_repair.py`.
- **New MCP tools** — add a tool to the MCP server (`test_mcp_server.py`) that proxies a new mind-service method.
- **New exporters** — add a sibling to `export_markdown` / `export_card` / `build_wiki`.
- **New derivation rules** — extend `derive`; each rule should be budget-aware (`derive_budget`).

## 10. Known Constraints

- **Requires Postgres** — the tests in `test_pg.py` / `test_pg_dispatch.py` are not trivially replaceable with SQLite; deployment assumes a real DB.
- **Async / concurrent status recomputation** — `test_pubsub.py` and `test_backtracking.py` imply status updates are propagated asynchronously; concurrent writers must serialize via the pub/sub bus or row locks.
- **LLM interactions are budgeted** — `derive_budget` imposes a hard cap, so any new LLM-using feature must respect it or deadlock the pipeline.
- **Content-addressing implies immutability** — once Merkle-hashed, a belief's content identity is fixed; edits create new records, which is why `deduplication` and `migrate_defeaters` exist.
- **MCP deployment assumes a trusted local process** — no evidence of auth on the MCP server, only `access_tags` on the data.

## Topics to Explore

- [file] `tests/conftest.py` — Contains the shared fixtures; reveals how the DB, LLM, and storage layer are constructed and torn down, which is the fastest way to learn the real module boundaries.
- [file] `tests/test_mind_service.py` — Central orchestration façade; shows which services the mind service composes and in what order.
- [file] `tests/test_dialectical.py` — The semantic heart of the defeasible/argumentation layer; the IN/OUT status rules live here.
- [file] `tests/test_nogood_id.py` — Exposes the nogood/assumption handles that tie the TMS to constraint-style reasoning.
- [file] `tests/test_pg_dispatch.py` — Shows how queries are routed to SQL; the place to look before adding any indexed lookup.
- [general] `mcp-tool-surface` — Inventory of MCP tools/resources exposed by `test_mcp_server.py` and how they map to mind-service operations.

## Beliefs

- `outlist-implies-tms-semantics` — Justifications carry an **outlist** (premises required to be OUT), confirming this is a Doyle-style TMS rather than a plain dependency graph.
- `mind-service-orchestrates-interfaces` — The CLI, HTTP API, and MCP server all route through the mind service; none talk to storage directly (`test_mind_service.py` plus the three interface test files).
- `storage-requires-postgres` — Persistence tests target a real Postgres backend (`test_pg.py`, `test_pg_dispatch.py`), so deployment requires a Postgres instance.
- `merkle-enables-content-deduplication` — Belief content is Merkle-hashed (`test_merkle.py`), which is what enables deduplication (`test_deduplication.py`) and integrity checks (`test_dependents_integrity.py`).
- `hf-import-defaults-to-configured-org` — HuggingFace imports resolve to a configurable default org when none is supplied (`test_hf_default_org.py`).
- `derive-enforces-budget` — Forward derivation is bounded by a configurable budget (`test_derive_budget.py`), so any new derivation rule must be budget-aware.
