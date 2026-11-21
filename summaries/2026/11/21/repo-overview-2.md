---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Repo Overview

# `reasons_lib` — Repository Overview

> Note: this overview is reconstructed from the module tree and entry-point hint. Where the actual source differs, treat the file names as the source of truth. The structure itself is highly communicative: this is a **belief / reason maintenance library** intended to sit behind an LLM agent (or as an MCP server) and give it a persistent, auditable knowledge base.

## 1. Purpose

`reasons_lib` is a **belief management and reasoning engine**. It stores assertions ("beliefs") together with their **premises** and **justifications**, allows new beliefs to be **derived** from existing ones, detects **contradictions**, **verifies** and **repairs** the graph, and presents the result through multiple **export** surfaces (wiki, markdown, cards, network graph).

The problem it solves is the one every agentic system eventually hits: an LLM can *state* things but has no durable, inspectable, revisable set of claims it is committed to. `reasons_lib` provides exactly that layer — a truth-maintenance substrate the model can query (`ask`), mutate (`propose_update`, `report_belief`), curate (`review`, `cluster`, `compact`), and publish (`build_wiki`, `export_markdown`).

## 2. Architecture

The library is layered, with the seams expressed in the top-level module split:

```
        ┌──────────────────────────────────────────────────────┐
        │  Surfaces      cli.py   api.py   mcp_server.py       │
        ├──────────────────────────────────────────────────────┤
        │  Workflows     ask, propose_update, report_*         │
        │                review_*, repair_*, derive, cluster,  │
        │                compact, contradictions, verify,      │
        │                check_stale, build_wiki, export_*     │
        ├──────────────────────────────────────────────────────┤
        │  Domain         metadata, network, mind_service, hf  │
        ├──────────────────────────────────────────────────────┤
        │  Infrastructure storage, pg, pubsub, llm, mcp_client │
        └──────────────────────────────────────────────────────┘
```

Key decisions apparent from this shape:

- **Storage is abstracted.** `storage.py` is a façade; `pg.py` is the concrete Postgres backend. Everything above talks to `storage`, nothing above talks to `pg` directly.
- **LLM is a single choke point.** `llm.py` (with `hf.py` for the Hugging Face path) is the only place model calls live. This is what lets `verify`, `derive`, `review_justifications`, and `contradictions` all be "LLM-assisted" without each one owning its own provider.
- **Dual protocol exposure.** `mcp_client.py` lets `reasons_lib` *consume* external MCP servers; `mcp_server.py` lets it *expose itself* to agent runtimes. Same library, two directions of the protocol.
- **Command-per-file pattern.** Nearly every workflow module maps 1:1 to a CLI subcommand. The CLI in `cli.py` is effectively a dispatcher over this directory.
- **Everything is replayable.** Because inputs (beliefs, premises, updates) are persisted and transformations (`derive`, `repair`, `compact`) write back to storage, the belief graph is an audit trail — nothing is "side-channel."

## 3. Key Components

**Surfaces**
- `cli.py` — primary entry point; dispatches subcommands to the workflow modules.
- `api.py` — programmatic/Python façade (and likely HTTP if deployed as a service).
- `mcp_server.py` / `mcp_client.py` — MCP protocol endpoints (server hosts the belief DB; client pulls resources from other MCP tools).

**Core belief operations**
- `report_belief.py` — ingest a single belief from a caller.
- `propose_update.py` — propose a change (add/retract/revise) subject to review.
- `derive.py` — infer new beliefs from existing premises + justifications.
- `contradictions.py` — find mutually inconsistent beliefs.
- `verify.py` — validate a belief against its supports.
- `repair.py` / `repair_premises.py` — resolve broken justifications or unsupported premises.

**Curation**
- `review.py`, `review_justifications.py`, `review_premises.py`, `report_gated.py` — human/agent-in-the-loop review gates.
- `cluster.py` — group semantically related beliefs.
- `compact.py` — collapse redundant beliefs.
- `check_stale.py` — flag beliefs past a freshness window.

**Ingestion / Egress**
- `import_agent.py`, `import_beliefs.py` — bring in beliefs from an agent or a batch.
- `export_card.py`, `export_markdown.py`, `build_wiki.py` — publish the belief base.
- `network.py` — build the belief/premise graph for downstream graph consumers.

**Infrastructure**
- `storage.py` — persistence façade.
- `pg.py` — Postgres binding (relational store for beliefs, premises, justifications, reviews).
- `pubsub.py` — change feed (likely so agents can subscribe to belief updates).
- `llm.py` / `hf.py` — model access.
- `metadata.py` — schema/versioning/annotations for beliefs.
- `mind_service.py` — the orchestration service that wires the above into a long-running process.

## 4. Data Flow

A canonical round-trip:

```
import_agent / import_beliefs / report_belief / mcp_server.receive
        │
        ▼
   storage (pg)  ◄── metadata (schema, provenance)
        │
        ▼
   derive / cluster / compact / contradictions / verify / repair
        │
        ▼
   review / review_premises / review_justifications / report_gated
        │
        ▼
   export_markdown / export_card / build_wiki / network
        │
        ▼
   ask / api / mcp_server response
```

In other words: **ingest → persist → reason → gate → publish → query**. Every stage writes back to `storage`, so the graph is always the source of truth (not an in-memory model), and `pubsub` broadcasts the deltas so subscribed agents observe the same state.

## 5. Dependencies

The notable external integrations implied by the file list:

| Module | Likely dependency | Why |
|---|---|---|
| `pg.py` | `psycopg` / `asyncpg` | Durable belief store; a graph of beliefs wants transactions and indexable relationships |
| `llm.py` | Any LLM SDK (OpenAI/Anthropic/generic) | Central abstraction so model choice is config, not code |
| `hf.py` | `huggingface_hub` / `transformers` | Alternative model execution path (local or hosted) |
| `mcp_client.py` | `mcp` Python SDK | Agent-tool interoperability |
| `mcp_server.py` | `mcp` Python SDK (server side) | Expose the belief base to agents |
| `pubsub.py` | Redis / Postgres LISTEN / NATS | Cross-process belief-change notifications |
| `network.py` | `networkx` (likely) | Graph algorithms over premises/justifications |
| `api.py` | `fastapi`/`starlette` (likely) | HTTP surface |

## 6. Entry Points

- **Primary:** `cli.py` — `python -m reasons_lib.cli <subcommand>` (or a configured `reasons` console script). Every workflow module backs a subcommand.
- **Secondary:** `api.py` — WSGI/ASGI app for HTTP deployments.
- **Secondary:** `mcp_server.py` — the process an agent runtime connects to over MCP.
- **Service loop:** `mind_service.py` — long-lived orchestrator that combines pubsub + review + derive into a background brain.

## 7. Configuration

Not visible from the tree alone, but implied:

- `metadata.py` — belief schema and versioning knobs.
- `llm.py` / `hf.py` — model provider, base URL, token, temperature, timeout (read from env or a config object).
- `pg.py` — DSN and pool sizing.
- `pubsub.py` — broker URL and channel naming.
- `check_stale.py` — staleness window/clock.
- `review*.py` — gating policy (auto-accept thresholds, required reviewers).

The design favors **env-driven configuration** because MCP servers and CLIs both need to start without a config file.

## 8. Module Boundaries

The layer diagram implies these import rules:

- **Surfaces** (`cli`, `api`, `mcp_server`) may import workflows; workflows may *not* import surfaces (no circular CLI dependency).
- **Workflows** may import domain + infrastructure. They must not reach into `pg.py` directly — they go through `storage.py`.
- **Infrastructure** (`llm`, `pg`, `pubsub`, `storage`, `mcp_client`) is a dependency sink: it imports only stdlib and third-party clients.
- **`metadata.py`** is referenced by storage/workflows but doesn't import workflows.
- **`mcp_client` and `mcp_server` are independent** — a process can be a consumer, a producer, or both, but the client must not drag the server (and vice versa).

This is what makes the library testable: you can swap `storage` for an in-memory fake, swap `llm` for a stub, and exercise every workflow offline.

## 9. Extension Points

- **New importers** — add a module alongside `import_beliefs.py` and register a CLI subcommand in `cli.py`.
- **New exporters** — add a module mirroring `export_markdown.py`; it should accept a storage handle and a selection predicate.
- **New repair strategies** — plug into `repair.py` / `repair_premises.py`; the pattern is "detect → propose → gate via `review`".
- **New LLM provider** — implemented once in `llm.py`; nothing upstream needs to change.
- **New storage backend** — implement the `storage` interface (see `pg.py` for the contract) and inject it.
- **New MCP tools** — add to `mcp_server.py`'s tool registry; ideally as thin wrappers over existing workflows so behavior matches the CLI.

## 10. Known Constraints

- **Postgres is a hard requirement** for the default configuration — `pg.py` is the only concrete storage backend in-tree.
- **External broker** may be required for `pubsub` if you want cross-process notifications; single-process use can no-op it.
- **LLM round-trips are the latency floor** for `verify`, `derive`, `review_justifications`, and `contradictions`. Batch/replay modes are essentially mandatory for anything but toy datasets.
- **Concurrency model** is implied by the module shapes rather than stated: `pubsub` + `mind_service` suggests a single writer / many reader architecture with a background review queue, not fully parallel mutation of the belief graph.
- **MCP version pinning** matters — client and server must speak compatible MCP protocol revisions.
- **No `__init__.py` re-exports are guaranteed**; consumers should import submodules explicitly (`from reasons_lib.storage import ...`), which keeps import cost low and avoids accidental coupling.

## Topics to Explore

- [file] `reasons_lib/storage.py` — The abstraction every workflow depends on; confirms the pg-is-an-implementation-not-a-requirement boundary.
- [file] `reasons_lib/cli.py` — Reveals the full subcommand surface and therefore the intended user workflows.
- [file] `reasons_lib/derive.py` — The generator of new beliefs; shows how premises + justifications are combined and how `review` gates admission.
- [general] `belief-lifecycle` — Trace one belief through ingest → verify → review → repair → export to understand the round-trip.
- [file] `reasons_lib/mcp_server.py` — The agent-facing surface; check whether tools are thin wrappers over workflows or their own logic.

## Beliefs

- `storage-is-the-only-path-to-pg` — Workflow modules obtain persistence exclusively through `storage.py`; `pg.py` is not imported by workflows.
- `llm-centralizes-all-model-calls` — Every LLM-assisted workflow (`verify`, `derive`, `review_justifications`, `contradictions`, `compact`) routes model access through `llm.py`.
- `mcp-server-enables-agent-access` — `mcp_server.py` exposes the belief base to agent runtimes over MCP, making the library usable as a tool endpoint.
- `mcp-client-enables-external-resource-ingest` — `mcp_client.py` is the counterpart that pulls beliefs/resources from other MCP servers, used by importer workflows.
- `derive-produces-new-beliefs` — `derive.py` writes newly inferred beliefs into `storage` rather than returning transient results, so derivations are auditable.
- `review-modules-gate-belief-admission` — The `review*.py` and `report_gated.py` modules enforce a curation gate before beliefs become first-class, distinguishing `report_belief` (immediate) from `propose_update` (gated).
