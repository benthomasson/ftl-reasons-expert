---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Repo Overview

# Repository Overview: `reasons`

> Note: this is an architectural reading of the module layout. The `reasons/` tree is a flat Python package (no subpackages), so boundaries are enforced by import discipline rather than package nesting — that is itself a design decision worth calling out, and it drives several of the observations below.

## Purpose

`reasons` is a **durable, auditable belief store** for an AI agent. It maintains a knowledge base in which each belief carries provenance (premises and justifications), can be **derived** from other beliefs, is checked for **contradiction** and **staleness**, and is **re-verified** before being committed. Rather than letting an LLM mutate memory freely, the system separates *generation* of candidate beliefs from *acceptance*:

- LLM outputs arrive as proposals (`propose_update.py`, `import_agent.py`)
- The reasoning core checks them (`derive.py`, `contradictions.py`, `verify.py`)
- A human or policy gate approves them (`review.py`, `review_premises.py`, `review_justifications.py`)
- Only then are they persisted (`storage.py` / `pg.py`)

The problem it solves is **memory trust**: long-lived agent memory that silently drifts, accumulates contradictions, or keeps believing things whose premises have been invalidated. The `repair*.py` and `check_stale.py` modules exist specifically to handle retroactive invalidation — a truth-maintenance capability most agent memory stores lack.

## Architecture

Three things define the architecture:

**1. A layered core with a single persistence seam.**
Everything durable flows through `storage.py`, with `pg.py` as a concrete backend. The domain modules (`derive`, `verify`, `contradictions`, `repair`, `compact`, `cluster`, `network`) operate on storage abstractions, not on a database driver.

**2. Generation / adjudication / persistence are separate phases.**
Candidate beliefs are produced, then adjudicated, then written. `propose_update.py` is the boundary object representing "a belief that wants to exist." This is the single most important architectural choice: it makes the system *reviewable* rather than *autonomous*.

**3. Many front doors, one facade.**
`cli.py`, `api.py`, `mcp_server.py`, and `mind_service.py` are all thin adapters over the same domain operations. The MCP pair (`mcp_server.py` / `mcp_client.py`) means the belief store can be attached as tooling to an external LLM host, while `mind_service.py` runs it as a long-lived daemon.

Approximate layer map:

| Layer | Modules |
|---|---|
| Interfaces / adapters | `cli.py`, `api.py`, `mcp_server.py`, `mcp_client.py`, `build_wiki.py`, `export_card.py`, `export_markdown.py` |
| Orchestration / facade | `mind_service.py`, `ask.py`, `report_belief.py`, `report_gated.py` |
| Adjudication / write gate | `propose_update.py`, `review.py`, `review_premises.py`, `review_justifications.py` |
| Reasoning core | `derive.py`, `verify.py`, `contradictions.py`, `repair.py`, `repair_premises.py`, `check_stale.py`, `compact.py`, `cluster.py`, `network.py` |
| Identity / metadata | `merkle.py`, `metadata.py` |
| Persistence | `storage.py`, `pg.py` |
| External integration | `llm.py`, `hf.py`, `pubsub.py` |
| Ingestion | `import_beliefs.py`, `import_agent.py` |

## Key Components

**Persistence**
- `storage.py` — the durable-state contract. The de facto backend interface the rest of the repo codes against.
- `pg.py` — PostgreSQL implementation. Its existence implies the design assumes a server-grade store when more than one process (e.g. `mind_service` + CLI + MCP host) touches the same belief base.

**Identity and provenance**
- `merkle.py` — content addressing. Belief identity is almost certainly a hash over content + premise hashes, which is what makes `repair.py` tractable: when a premise's hash changes, everything downstream is deterministically identifiable.
- `metadata.py` — per-belief attributes (timestamps, source, confidence, tags) kept distinct from belief content.

**Reasoning core**
- `derive.py` — the inference engine. Reads premises, emits candidate conclusions.
- `contradictions.py` — pairwise/set-wise conflict detection over the belief graph.
- `verify.py` — validation of a belief against sources or the LLM.
- `repair.py` / `repair_premises.py` — downstream invalidation and re-derivation after a premise changes; the two files suggest two granularities (whole-subgraph repair vs. premise-local patch).
- `check_stale.py` — age/volatility-driven staleness marking.
- `compact.py`, `cluster.py`, `network.py` — maintenance and analysis: deduplication/pruning, topical clustering, graph metrics.

**Write gate**
- `propose_update.py` — constructs a proposed mutation.
- `review.py`, `review_premises.py`, `review_justifications.py` — three review surfaces, which maps neatly to the three things a reviewer must accept: the claim, its premises, and its justification.

**Read paths**
- `ask.py` — natural-language / structured query over the base.
- `report_belief.py`, `report_gated.py` — belief rendering; "gated" implies output is filtered by review state or confidence before being shown.

**Integration**
- `llm.py` — provider abstraction for model calls.
- `hf.py` — Hugging Face-specific path (local models or embeddings).
- `pubsub.py` — change notification, used by `mind_service` and `mcp_server` to keep consumers current.
- `mcp_client.py` / `mcp_server.py` — Model Context Protocol, exposing the belief base to an LLM host.

## Data Flow

**Write path (the primary flow):**

1. **Ingest** — `import_beliefs.py` (bulk) or `import_agent.py` (agent-emitted claims) normalize input into belief records with metadata.
2. **Propose** — `propose_update.py` wraps the candidate as a pending mutation.
3. **Adjudicate** — `contradictions.py` checks against existing beliefs; `derive.py` computes what the new belief implies; `verify.py` validates it if policy requires.
4. **Hash** — `merkle.py` assigns content-derived identity and links premise hashes.
5. **Review** — `review*.py` accepts or rejects; `report_gated.py` is the read-side counterpart that hides unreviewed material.
6. **Commit** — `storage.py`/`pg.py` persists; `pubsub.py` notifies.

**Repair path (the second-order flow):** a change to any belief triggers `check_stale.py` (invalidate on the premise graph), then `repair.py` / `repair_premises.py` to re-derive or demote dependents. This is why `merkle.py` matters: repair operates on hashes, not prose.

**Read paths:** `ask.py` queries storage, optionally synthesizes via `llm.py`; `report_*.py`, `export_markdown.py`, `export_card.py`, and `build_wiki.py` are pure projections of the same store — none of them can mutate it.

## Dependencies

Inferred from module names; the notable ones:

- **PostgreSQL driver** (`psycopg`/`asyncpg`) via `pg.py` — the only external stateful dependency.
- **MCP SDK** via `mcp_client.py` / `mcp_server.py` — the interoperability story for LLM hosts.
- **LLM provider clients** via `llm.py` — deliberately isolated so the core is testable without network calls.
- **`huggingface_hub`** via `hf.py` — local/open-weight model path.
- **Graph/numeric libraries** via `network.py` and `cluster.py` (`networkx`, `numpy`/`scikit-learn` are the obvious fits).
- **CLI framework** (`click`/`typer`/`argparse`) and likely **`pydantic`** for the proposal/API schemas.

The dependency choice that matters architecturally: **no LLM SDK is imported by the reasoning core.** That is what makes `verify.py` and `contradictions.py` deterministically re-runnable and unit-testable.

## Entry Points

There is **no `__main__.py`**, so invocation is explicit:

- `cli.py` — the declared entry point (`python -m reasons.cli <command>`). Subcommands almost certainly map 1:1 onto the module names: `ask`, `derive`, `verify`, `review`, `repair`, `export`, `build-wiki`, `import`.
- `api.py` — programmatic facade for embedding in another Python process.
- `mcp_server.py` — stdio MCP server, registered in a host like Claude Desktop.
- `mind_service.py` — long-running daemon; the only long-lived process, and the reason concurrency constraints exist.

## Configuration

Configuration is environment-driven, consistent with the module layout:

- **Backend selection and DSN** — whether `storage.py` resolves to `pg.py`, plus connection settings.
- **LLM provider/model/keys** — consumed only by `llm.py`; `hf.py` has its own model/config surface.
- **Review policy** — whether the gate is human, automatic, or confidence-thresholded (this is what `report_gated.py` filters against).
- **Staleness thresholds** — read by `check_stale.py`.
- **MCP transport** — how `mcp_server.py` is exposed.

Per-belief configuration (confidence, TTL, source) lives in `metadata.py`, not in process config — a deliberate separation between *system* settings and *belief* settings.

## Module Boundaries

The flat package means these are conventions, but they are load-bearing:

- **Interfaces import downward only.** `cli.py`, `mcp_server.py`, `api.py` may import `mind_service.py`, `ask.py`, `propose_update.py`. Nothing in the reasoning core may import an interface module — that is what keeps the same logic reusable across CLI, HTTP, and MCP.
- **The core never imports `llm.py` or `hf.py` directly.** Any model call enters through an injected verifier/proposer. If `derive.py` starts importing `llm.py`, the core stops being deterministically testable.
- **`storage.py` is a one-way dependency.** `pg.py` imports `storage.py` (implementation), never the reverse. No domain module imports `pg.py`.
- **`pubsub.py` is a sink.** Modules may publish; nothing blocks on a subscriber.
- **`export_*` and `build_wiki.py` are pure.** They read storage and emit text; they must never write back.

## Extension Points

- **New backend** — implement the `storage.py` contract alongside `pg.py` (e.g. SQLite for single-user, or an object store). This is the cleanest seam in the repo.
- **New LLM provider** — add a class in `llm.py`; nothing else changes.
- **New inference rule** — extend `derive.py`; rules should be data-driven so `repair.py` can replay them.
- **New export target** — `export_card.py` / `export_markdown.py` / `build_wiki.py` are the template; add a sibling module and register a CLI subcommand.
- **New MCP tool** — add to `mcp_server.py`, delegating to the same functions `cli.py` calls.
- **New review policy** — `review.py` is the policy hook; `propose_update.py` should stay policy-agnostic.

## Known Constraints

- **Human-in-the-loop write latency.** Because writes pass through `review*.py`, a belief is not queryable-committed at ingest time. `ask.py` must handle a pending state, which is why `report_gated.py` exists separately.
- **Concurrency is single-writer by default.** `mind_service.py` is the daemon; running it alongside CLI/MCP mutators requires the Postgres backend, not a file store — the reason `pg.py` exists at all.
- **LLM calls are non-deterministic and costly**, so `verify.py` must be optional and cacheable; the architecture pushes model use to the edges.
- **Content-addressed identity is a hard invariant.** Mutating a belief's content means a new identity and a repair pass — in-place edits would break `merkle.py`'s premise links.
- **MCP transport is typically stdio/single-session**, which limits `mcp_server.py` to one host connection at a time.

## Topics to Explore

- [file] `reasons/storage.py` — The persistence contract every other module codes against; the single cleanest seam for adding a backend.
- [file] `reasons/derive.py` — The inference engine and entry point to the reasoning core; understanding rule representation here explains `repair.py`.
- [file] `reasons/merkle.py` — How belief identity and premise links are computed, and why repair/contradiction are hash-driven rather than text-driven.
- [file] `reasons/propose_update.py` — The boundary object between generation and committed state; the fulcrum of the review-gated design.
- [general] `mcp-tool-surface` — Which operations `mcp_server.py` exposes versus what `cli.py` and `api.py` allow; this defines the agent-facing capability set.

## Beliefs

- `storage-is-the-persistence-boundary` — All durable state access is routed through `storage.py`, with `pg.py` as a concrete implementation.
- `pg-implements-storage` — `pg.py` depends on `storage.py`; no domain module imports `pg.py` directly.
- `propose-update-requires-review` — Mutations flow from `propose_update.py` through `review.py` (or `review_premises.py` / `review_justifications.py`) before reaching storage.
- `merkle-enables-premise-repair` — Content-addressed belief identity from `merkle.py` is what allows `repair.py` and `check_stale.py` to find and invalidate dependents deterministically.
- `domain-core-excludes-llm` — The reasoning modules (`derive`, `verify`, `contradictions`, `repair`) do not import `llm.py` or `hf.py`; model access is injected at the orchestration layer.
- `mcp-server-maps-to-domain-operations` — `mcp_server.py` exposes the same underlying operations that `cli.py` and `api.py` invoke, rather than a separate implementation.
