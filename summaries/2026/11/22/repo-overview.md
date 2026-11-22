---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Repo Overview

# reasons_lib — Repository Overview

## 1. Purpose

`reasons_lib` is a **belief/knowledge-base engine with an epistemic reasoning model**. Rather than storing plain notes, it stores *beliefs* that are nodes in a graph: each belief can have **premises** (what it depends on), **justifications** (why it is held), and **derivations** (what follows from it). On top of that graph it runs a set of reasoning and hygiene workflows:

- **Derivation & verification** — `derive.py`, `verify.py`
- **Contradiction & staleness detection** — `contradictions.py`, `check_stale.py`
- **Repair & review loops** — `repair.py`, `repair_premises.py`, `review.py`, `review_justifications.py`, `review_premises.py`, `propose_update.py`
- **Clustering & compaction** — `cluster.py`, `compact.py`
- **Import/export & presentation** — `import_beliefs.py`, `import_agent.py`, `export_card.py`, `export_markdown.py`, `build_wiki.py`, `report_belief.py`, `report_gated.py`

The problem it solves: maintaining a *consistent, auditable* body of knowledge where change to one belief propagates (via premises) and must be re-reviewed, rather than a flat document that silently drifts.

## 2. Architecture

A **flat, single-package layout** (`reasons_lib/*.py`, no subpackages). Rather than enforcing layering structurally, the codebase organizes by *concern* and relies on a **facade** (`api.py`) plus **abstraction modules** (`storage.py`, `llm.py`) to keep boundaries.

Conceptual layers:

| Layer | Modules | Role |
|---|---|---|
| Domain / reasoning core | `network.py`, `derive.py`, `contradictions.py`, `cluster.py`, `verify.py`, `metadata.py` | Graph model of beliefs, inference, consistency checks |
| Persistence | `storage.py`, `pg.py`, `compact.py` | Storage abstraction; Postgres backend; compaction of history/state |
| LLM & agent bridge | `llm.py`, `hf.py`, `ask.py`, `import_agent.py` | Model provider abstraction; querying; ingesting agent output |
| Workflows (human/LLM-in-the-loop) | `review*.py`, `repair*.py`, `propose_update.py`, `check_stale.py` | Stateful multi-step processes over beliefs |
| Reporting & export | `report_belief.py`, `report_gated.py`, `export_card.py`, `export_markdown.py`, `build_wiki.py` | Read-only projections of the graph |
| Interfaces | `cli.py`, `api.py`, `mcp_server.py`, `mcp_client.py`, `mind_service.py`, `pubsub.py` | Entry surfaces and eventing |

**Design patterns:**

- **Facade** — `api.py` is the public programmatic surface; `cli.py` and `mcp_server.py` are thin adapters over it.
- **Port/adapter for storage** — `storage.py` defines the interface; `pg.py` is the Postgres adapter.
- **Provider abstraction for LLMs** — `llm.py` centralizes model calls so `derive.py`, `review*.py`, `ask.py`, `repair.py` are provider-agnostic; `hf.py` is a HuggingFace-specific path.
- **Pipeline/workflow** — the `review_*`, `repair_*`, `propose_update`, `derive`, `verify` files are discrete pipeline stages that can be composed.
- **Protocol adapter (MCP)** — `mcp_server.py`/`mcp_client.py` expose the library as tools/resources over the Model Context Protocol.

## 3. Key Components

- **`network.py`** — The belief graph: nodes (beliefs) and edges (premise/justification/derivation relationships). Likely the canonical data model other modules import.
- **`storage.py` / `pg.py`** — `storage.py` is the interface/store; `pg.py` implements Postgres persistence. All workflows read/write through here, not direct SQL.
- **`llm.py`** — Single LLM client abstraction; the choke point for model selection, retries, prompts.
- **`derive.py` / `verify.py`** — Forward reasoning (generate new believed statements from premises) and verification of existing beliefs against evidence.
- **`contradictions.py` / `check_stale.py`** — Consistency maintenance: find conflicting beliefs and premises whose support has eroded.
- **`repair.py` / `repair_premises.py`** — Remediation: rewrite/fix broken premise chains.
- **`review*.py`** — Approval workflow stages: review a belief, its justifications, its premises.
- **`cluster.py` / `compact.py`** — Redundancy reduction: group similar beliefs and compact history/state.
- **`ask.py`** — Query interface over the belief graph (likely LLM-assisted retrieval + synthesis).
- **`api.py`** — Public façade tying the above together for `cli.py` and `mcp_server.py`.
- **`mcp_server.py` / `mcp_client.py` / `mind_service.py` / `pubsub.py`** — Runtime/serving surfaces: expose the graph as an MCP server, call external MCP tools, run the "mind" service loop, and publish events.
- **`import_beliefs.py` / `import_agent.py` / `hf.py`** — Ingest beliefs from files/agents/HuggingFace.
- **`export_*.py` / `build_wiki.py` / `report_*.py`** — Output projections (cards, markdown, a wiki, per-belief reports).
- **`metadata.py`** — Provenance/versioning metadata attached to beliefs.
- **`cli.py`** — Command entry point.

## 4. Data Flow

```
import_beliefs / import_agent / hf
        │  parse → belief records
        ▼
   storage.py / pg.py  ◄──────────────┐
        │  graph load                  │
        ▼                              │
   network.py (belief graph)           │
        │                              │
        ├─ derive.py ──► new beliefs ──┤
        ├─ contradictions.py ──► flags │
        ├─ check_stale.py ──► flags    │
        │                              ▼
        ├─ review*.py ──► approve/reject ── propose_update / repair*
        │                              │
        ▼                              │
   ask.py / report_*.py / export_* / build_wiki ──► outputs
        │
        └─ mcp_server.py / mind_service.py / pubsub.py ──► external consumers
```

1. **Ingest** via `import_*` or direct `api.py` calls.
2. **Persist** through `storage.py` (Postgres via `pg.py`).
3. **Reason** in `network.py` + `derive.py`, producing new beliefs and dependency edges.
4. **Audit** with `contradictions.py`, `check_stale.py`, `verify.py`, emitting actionable findings.
5. **Remediate** through `review*.py` / `repair*.py` / `propose_update.py`, which may loop back into storage.
6. **Serve/export** through `ask.py`, `report_*.py`, `export_*.py`, `build_wiki.py`, or the MCP/`mind_service` interfaces.

Because beliefs are graph nodes with premise edges, every write can invalidate downstream beliefs — that is why the review/repair workflows sit between the reasoning core and the output surfaces.

## 5. Dependencies

Given the module names, the notable external dependencies are:

- **LLM provider SDK(s)** — used by `llm.py` (and `hf.py` for HuggingFace), underpinning `derive.py`, `ask.py`, `review*.py`, `repair*.py`, `cluster.py`. This is the single heaviest dependency and the main source of non-determinism.
- **PostgreSQL driver** — used by `pg.py` for the persistent store; `storage.py` abstracts it so an alternative backend can be slotted in.
- **MCP SDK** — used by `mcp_server.py` and `mcp_client.py` to expose/consume the belief graph over the Model Context Protocol.
- **Pub/sub backend** — referenced by `pubsub.py` for event distribution (likely Redis or an in-process bus).
- **HuggingFace tooling** — `hf.py`.

## 6. Entry Points

- **`cli.py`** — The declared entry point; subcommands likely map one-to-one onto workflows (`derive`, `review`, `repair`, `export`, `ask`, `serve`, etc.).
- Secondary invocation surfaces (likely invoked via `cli.py` subcommands or `python -m`):
  - `mcp_server.py` — run as an MCP server.
  - `mind_service.py` — long-running service.
  - `api.py` — imported directly by external code.

## 7. Configuration

Configuration is naturally concentrated in the abstraction layers:

- **Storage** — DB connection settings consumed by `pg.py` (host, database, credentials).
- **LLM** — model choice, API keys, timeouts, provider selection read by `llm.py` (and `hf.py`).
- **Runtime/serving** — MCP transport, service bindings, pubsub endpoints in `mcp_server.py`/`pubsub.py`/`mind_service.py`.
- **Metadata** — `metadata.py` records provenance/schema versioning so exports and imports stay compatible.

Because there is no `config.py` or `settings.py`, configuration is almost certainly environment/CLI-driven per module.

## 8. Module Boundaries

The layout is intentionally flat, so boundaries are **by convention** rather than enforced by packages:

- **`storage.py` / `pg.py`** — The sole path to persistence. Workflows should not issue SQL directly.
- **`llm.py`** — The sole path to model calls. `derive.py`, `review*.py`, `repair*.py`, `ask.py` should not instantiate provider clients directly.
- **`network.py` + `metadata.py`** — The domain model; everything else may import it, and it should not import workflows.
- **`api.py`** — The façade: `cli.py`, `mcp_server.py`, and `mind_service.py` call into it; it orchestrates the lower layers.
- **`export_*` / `report_*` / `build_wiki`** — Read-only consumers; they should not mutate the graph.
- **`import_*` / `hf.py`** — Inbound adapters; they produce belief records and hand them to storage.

The intended dependency direction is roughly:

```
interfaces (cli, mcp_*, mind_service)
        ↓
api.py
        ↓
workflows (review*, repair*, derive, verify, cluster, compact)
        ↓
storage.py / llm.py / network.py
```

## 9. Extension Points

- **New storage backend** — implement the `storage.py` interface in a new module (e.g., a SQLite/JSON adapter) parallel to `pg.py`.
- **New LLM provider** — add a branch or module alongside `hf.py` behind the `llm.py` abstraction.
- **New workflow stage** — drop a new module next to `review_justifications.py` / `repair_premises.py` and wire it into `api.py` + a `cli.py` subcommand.
- **New export format** — add `export_<format>.py` following `export_card.py` / `export_markdown.py` and hook it into the CLI.
- **New ingestion source** — add `import_<source>.py` mirroring `import_beliefs.py` / `import_agent.py`.
- **New MCP tool** — register it in `mcp_server.py`, delegating to `api.py`.

## 10. Known Constraints

- **LLM-dependent determinism** — `derive.py`, `review*.py`, `repair*.py`, `cluster.py`, `ask.py` result quality/consistency depends on model behavior; runs are not reproducible bit-for-bit.
- **No enforced package layering** — because everything sits in one flat package, import cycles between workflows and the core are possible and must be avoided by discipline.
- **Single persistent backend in-tree** — Postgres (`pg.py`) is the only concrete storage adapter; switching requires a new `storage.py` implementation.
- **CLI-oriented** — `cli.py` is the single declared entry point; the service/MCP surfaces are secondary and likely dependent on the CLI or direct import.
- **Network/credential requirements** — LLM, HuggingFace, MCP, and pubsub modules all require credentials/network access; offline operation is limited to storage + core graph operations.
- **Graph-wide invalidation cost** — editing a premise can invalidate arbitrary downstream beliefs; review/repair workflows are necessary to keep the graph trustworthy and are potentially expensive.

## Topics to Explore

- [file] `reasons_lib/network.py` — The canonical belief/premise graph model that every other module builds on.
- [file] `reasons_lib/storage.py` — The persistence interface and how `pg.py` is expected to implement it.
- [file] `reasons_lib/api.py` — The orchestration façade that defines the actual public contract of the library.
- [function] `reasons_lib/llm.py` (entry symbol) — The single choke point through which all model calls pass.
- [general] `review-and-repair-workflow` — How `review*.py` and `repair*.py` stage, gate, and write back approved changes to the belief graph.

## Beliefs

- `flat-package-layout` — All modules live directly under the single `reasons_lib` package, with no nested subpackages.
- `cli-is-entry-point` — `cli.py` is the only declared entry point for the project.
- `storage-abstracts-pg` — `storage.py` defines the persistence interface that `pg.py` implements as the Postgres backend.
- `llm-centralizes-model-calls` — `llm.py` is the shared abstraction used by workflow modules (`derive.py`, `review*.py`, `ask.py`, `repair*.py`) instead of providers being called directly.
- `mcp-server-pairs-with-client` — `mcp_server.py` exposes the library over MCP and `mcp_client.py` consumes external MCP tools, forming the protocol boundary of the project.
