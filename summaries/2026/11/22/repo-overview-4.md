---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Repo Overview

# Repository Overview — `ftl-reasons` (Reasons)

## Purpose

`ftl-reasons` implements a **Truth Maintenance System (TMS)** in the tradition of Doyle (1979). It models a belief set as a dependency graph:

- Every **node** has a truth value of `IN` (believed) or `OUT` (retracted).
- Nodes are either **premises** (always IN) or **derived** (IN when at least one of their justifications is valid).
- A **justification** can be `SL` (support list — true when all antecedents are IN) or `CP` (conditional proof) and supports **outlists** ("believe X unless Y"), enabling non-monotonic reasoning.
- When a node retracts, everything downstream is re-evaluated (**retraction cascade**); when it restores, dependents come back (**restoration without rederivation**).
- A **nogood** records an impossible combination; the engine performs **dependency-directed backtracking** to retract the minimal responsible premise.
- On top of this it layers **dialectical argumentation** (`challenge` / `defend`), stale-source detection, markdown/wiki export, and a natural-language `ask` interface.

In short: a durable, SQLite-backed belief store with causal retraction semantics, wrapped in a `reasons` CLI, a Python API, and an MCP server.

## Architecture

The code is organized as a **layered application** with a small core and many optional adapters:

```
┌──────────────────────────────────────────────────────────┐
│  Interfaces         cli.py · mcp_server.py · api.py      │
├──────────────────────────────────────────────────────────┤
│  Services           mind_service.py · pubsub.py          │
│                     review*.py · report_*.py             │
├──────────────────────────────────────────────────────────┤
│  Engine             network.py  derive.py  contradictions│
│                     repair*.py  verify.py  merkle.py     │
├──────────────────────────────────────────────────────────┤
│  Persistence        storage.py (SQLite) · pg.py (Postgres)│
├──────────────────────────────────────────────────────────┤
│  Adapters           import_*.py  export_*.py  hf.py      │
│                     build_wiki.py  check_stale.py        │
│                     compact.py  cluster.py  llm.py  ask  │
└──────────────────────────────────────────────────────────┘
```

Key design decisions:

1. **A small dependency-free core** — `pyproject.toml` declares `dependencies = []`. SQLite is accessed through the standard library, so `reasons init` + basic `add`/`retract`/`explain` work with **no third-party packages**.
2. **Everything heavier is an extra** — Postgres, LLM, MCP, clustering, HuggingFace, and Langfuse are all gated behind optional extras (`pg`, `api`, `vertex`, `openai`, `llm`, `mcp`, `cluster`, `langfuse`).
3. **Two large duplicate packages** — `reasons/` and `reasons_lib/` mirror each other file-for-file except `reasons_lib/` omits `merkle.py` and `verify.py`. Both are installed (`packages = ["reasons", "reasons_lib"]`), and both entry points target `reasons.*`. This looks like a **rename/migration in progress** — `reasons/` is the canonical namespace (it has the Merkle/verify integrity primitives and owns the console scripts), while `reasons_lib/` is a parallel copy. Anyone editing core logic must update both trees.
4. **Separation of engine and storage** — the graph/belief logic lives in `network.py`; persistence is abstracted behind `storage.py` (SQLite) and `pg.py` (Postgres/`psycopg`). The CLI, API, MCP server, and services all go through these.
5. **Provenance is first-class** — nodes carry `source` metadata, source hashes (`merkle.py`, `metadata.py`), a propagation audit log, and cost artifacts (`costs/…json`), so stale inputs and reasoning history can be re-examined (`check_stale`, `trace`, `explain`).

## Key Components

### Core engine

| File | Responsibility |
|---|---|
| `reasons/network.py` | The dependency graph, truth-value propagation, retraction/restoration cascades. The heart of the TMS. |
| `reasons/derive.py` | Adds derived nodes with SL/CP justifications and drives re-evaluation. |
| `reasons/contradictions.py` | Records nogoods and resolves them via dependency-directed backtracking. |
| `reasons/repair.py`, `repair_premises.py` | Repair strategies when a nogood is found (choose which premise to retract). |
| `reasons/verify.py` | Verifies justifications / integrity of the network. |
| `reasons/merkle.py` | Content-addressable Merkle hashing for detecting state drift (only present in `reasons/`). |

### Persistence

| File | Responsibility |
|---|---|
| `reasons/storage.py` | SQLite persistence for nodes, justifications, dependents, nogoods, and the propagation log (the `reasons.db` file). |
| `reasons/pg.py` | Postgres backend (`psycopg`) for multi-process / hosted deployments. |
| `reasons/metadata.py` | Tracks source paths, hashes, tags, and other node metadata. |

### Interfaces

| File | Responsibility |
|---|---|
| `reasons/cli.py` | `reasons` console script. Subcommands: `init`, `add`, `retract`, `assert`, `status`, `show`, `explain`, `challenge`, `defend`, `nogood`, `trace`, `propagate`, `log`, `search`, `list`, `import-beliefs`, `export`, `export-markdown`, `check-stale`, `hash-sources`, `compact`. |
| `reasons/api.py` | Programmatic API surface used by the CLI and library consumers. |
| `reasons/mcp_server.py` | MCP (Model Context Protocol) server: `reasons-mcp`. Exposes belief operations as tools for LLM agents. |
| `reasons/mcp_client.py` | Client for talking to an MCP server. |
| `reasons/ask.py` | Natural-language Q&A over the belief graph (uses `llm.py`). |
| `reasons/mind_service.py` | Higher-level orchestration: manage a "mind"/agent belief state. |

### Adapters and utilities

| File | Responsibility |
|---|---|
| `reasons/import_beliefs.py` | Imports a `beliefs.md` registry (plus auto-detected `nogoods.md`). |
| `reasons/import_agent.py` | Imports from an agent/other representation. |
| `reasons/hf.py` | HuggingFace dataset import/export. |
| `reasons/export_markdown.py`, `export_card.py` | Render the graph as human-readable markdown or as a card. |
| `reasons/build_wiki.py` | Generates a wiki-style site from the belief graph. |
| `reasons/check_stale.py` | Compares stored source hashes with the filesystem to flag stale nodes. |
| `reasons/compact.py` | Token-budgeted summary generation (for context injection into an LLM). |
| `reasons/cluster.py` | Semantic clustering of nodes (`sentence-transformers`, `scikit-learn`). |
| `reasons/llm.py` | Provider abstraction over LangChain Anthropic/Vertex, OpenAI, Langfuse tracing. |
| `reasons/propose_update.py`, `report_belief.py`, `report_gated.py` | Proposal / proposal-review and structured report generation. |
| `reasons/review*.py` | Review queues for premises, justifications, and general nodes. |
| `reasons/pubsub.py` | Event bus for propagating changes to subscribers (likely Redis-like in `pg` mode). |

### Ancillary

- `.claude/skills/reasons/SKILL.md` — a Claude skill describing how to drive the CLI.
- `.forge/proposed-entries.json`, `.forge/report.json` — artifacts from a "forge" curation tool.
- `costs/2026-11-19T180331-update.json` — recorded cost/provenance for an update run, indicating that LLM-backed operations are tracked for spend.
- `.github/workflows/tests.yml` — CI runs the pytest suite.

## Data Flow

**Interactive (CLI):**

```
user → reasons <cmd> → cli.main → api.py or storage directly
       → storage.py (SQLite) writes node/justification
       → network.py recomputes truth values via propagation queue
       → storage.py persists the log entries
       → cli prints status / explain / cascade summary
```

**Import/Export:**

```
beliefs.md  ──parse──►  storage (nodes + justifications + nogoods)
                              │
                              ├─► network.py propagation
                              └─► export_markdown / export_card / build_wiki
```

**Reasoning-tool (MCP / LLM):**

```
agent → reasons-mcp (mcp_server.main)
      → api.ask / mind_service
      → llm.py (LangChain / OpenAI)
      → api.py mutations against the graph
      → pubsub for subscribers
```

**Contradiction handling:**

```
nogood A B  ──record──►  contradictions.py
                        → backtrack justification graph
                        → repair.py selects responsible premise
                        → network.py retracts + cascades
                        → log
```

The **canonical output artifacts** are: nodes with `IN`/`OUT` truth values (SQLite `reasons.db`), node justifications (SL/CP with inlist/outlist), nogoods, and a propagation log; plus exported `beliefs.md`, JSON, wiki, cards, and token-budgeted summaries.

## Dependencies

**Runtime (required):** none — `[project.dependencies] = []`. SQLite is accessed via `sqlite3` from the standard library. This makes the core installable offline and trivially testable.

**Optional extras:**

| Extra | Packages | Used by |
|---|---|---|
| `pg` | `psycopg[binary]` | `pg.py` (Postgres backend) |
| `test` | `pytest`, `pytest-cov`, `pyyaml` | the test suite |
| `cluster` | `sentence-transformers`, `scikit-learn` | `cluster.py` |
| `mcp` | `mcp>=1.0` | `mcp_server.py`, `mcp_client.py` |
| `api` | `langchain-anthropic` | `llm.py` (Anthropic path) |
| `vertex` | `langchain-google-vertexai` | `llm.py` (Vertex path) |
| `openai` | `openai` | direct OpenAI calls |
| `langfuse` | `langfuse` | tracing |
| `llm` | LangChain Anthropic + Vertex + Langfuse | aggregate LLM extras |
| `test-pg` | psycopg + test tooling | Postgres tests |

## Entry Points

Defined in `pyproject.toml` under `[project.scripts]`:

- `reasons` → `reasons.cli:main` — the primary CLI described in the README.
- `reasons-mcp` → `reasons.mcp_server:main` — an MCP JSON-RPC server that exposes belief operations as tools to LLM clients.

Library use: `import reasons` and call into `reasons.api` / `reasons.mind_service`. The dual-package layout also makes `reasons_lib` importable, though it is not wired to any script.

## Configuration

- **Optional extras** (above) — the primary knob for enabling features.
- **Storage location** — a SQLite file created by `reasons init` (`reasons.db` by default).
- **Postgres** — enabled by installing `[pg]` and configuring `pg.py` (likely via connection string / env vars).
- **LLM providers** — selected by installing `api`, `vertex`, or `openai` extras; `llm.py` dispatches on provider.
- **Langfuse tracing** — activated by installing the `langfuse` extra.
- **Source staleness** — node `source` paths and their hashes stored via `metadata.py`; `check-stale` compares against the filesystem.
- **Context/budget knobs** — `compact --budget N` controls token budget for summaries.

## Module Boundaries

The layered picture implies these rules (worth verifying in code):

- **`storage.py` / `pg.py`** know nothing about the CLI or LLM layers — they only see node/justification/nogood records.
- **`network.py`** is the engine and is imported by `cli.py`, `api.py`, `mcp_server.py`, `mind_service.py`, `derive.py`, `contradictions.py`, `repair*.py`. It should **not** import `cli`, `mcp_*`, `llm`, or `ask` — keeping the engine usable in library/offline contexts.
- **`import_*` / `export_*` / `build_wiki` / `check_stale` / `compact` / `cluster`** are leaf adapters: they depend on `storage`/`network` but nothing else in the graph depends on them.
- **`llm.py` / `ask.py`** sit behind the `llm`/`api`/`vertex`/`openai` extras and should be optional imports from `cli`. `mcp_server.py` is the natural caller.
- **`mcp_server.py`** is an *interface* module like `cli.py` — it must not be imported by core code.
- **`reasons/` ↔ `reasons_lib/`** — the two trees appear to be mirrors; the intent is presumably that both packages be kept in sync (with `reasons_lib` lagging by `merkle.py`/`verify.py`).

## Extension Points

- **New CLI subcommand** — add a parser in `reasons/cli.py`, route to `api.py`.
- **New storage backend** — mirror `pg.py`'s interface against `storage.py`; the engine only needs CRUD + log append.
- **New importer/exporter** — add `reasons/import_<format>.py` or `reasons/export_<format>.py`; test alongside `tests/test_import_*.py` / `tests/test_export_*.py`.
- **New justification kind or resolution strategy** — extend `network.py` (justification evaluation) and `contradictions.py`/`repair.py` (backtracking/repair selection).
- **New LLM provider** — add a dispatch branch to `llm.py` and a matching extra in `pyproject.toml`.
- **New MCP tool** — extend `mcp_server.py`; the tool surface is the natural place for agents to drive the TMS.
- **New review workflow** — add a `review_<thing>.py` sibling to the existing `review.py`, `review_premises.py`, `review_justifications.py`.

## Known Constraints

- **Python ≥ 3.10** required.
- **Single-process SQLite default** — safe for one CLI user; concurrent writers need the `pg` extra.
- **Optional deps are truly optional** — importing `ask`, `cluster`, `llm`, `mcp_server`, `hf`, `pg` will fail without the corresponding extra, so `cli.py` must guard those imports.
- **Two parallel package trees** — any change to core logic must be mirrored in `reasons_lib/`; `merkle.py` and `verify.py` currently exist only under `reasons/`, so `reasons_lib/` is not a drop-in superset.
- **No silent fallbacks for LLM paths** — without an LLM extra, `ask` and `propose_update` (if LLM-backed) cannot run; the README's CLI examples intentionally use only dependency-free commands.
- **Cost/provenance side effects** — `costs/*.json` and `.forge/*.json` suggest that certain operations (LLM updates, forge runs) write artifacts; deployments should treat these directories as state.
- **MCP as separate process** — `reasons-mcp` is its own entry point rather than being embedded in `reasons`, meaning an agent typically talks to the store over MCP (or via the `mind_service`).

## Topics to Explore

- [file] `reasons/network.py` — The propagation engine: cascade, restoration, and multi-justification semantics live here.
- [file] `reasons/contradictions.py` — How nogoods are recorded and how dependency-directed backtracking chooses a premise to retract.
- [file] `reasons/storage.py` — The SQLite schema and log-based persistence model; mirrors `pg.py`.
- [function] `reasons/cli.py:main` — The full subcommand surface and how it dispatches into the engine vs. adapters.
- [general] `reasons-vs-reasons-lib` — Why two parallel packages exist (rename? shim?), and whether `merkle.py`/`verify.py` are missing from `reasons_lib/` accidentally or by design.

## Beliefs

- `core-has-zero-required-runtime-dependencies` — `pyproject.toml` declares `dependencies = []` and all third-party packages are gated behind optional extras (`pg`, `llm`, `mcp`, `cluster`, `langfuse`, `openai`, `api`, `vertex`).
- `reasons-has-two-parallel-package-trees` — `reasons/` and `reasons_lib/` list the same module filenames, except `reasons_lib/` omits `merkle.py` and `verify.py`.
- `console-scripts-target-the-reasons-package` — Both `reasons` and `reasons-mcp` entry points in `[project.scripts]` resolve to modules inside `reasons/`, not `reasons_lib/`.
- `storage-implements-two-backends` — `storage.py` (SQLite via stdlib) and `pg.py` (Postgres via `psycopg`) provide the persistence layer behind the CLI and API.
- `extras-gate-optional-capabilities` — Every non-stdlib capability (Postgres, LLM, MCP, clustering, Langfuse tracing) is installable only via a named extra in `pyproject.toml`.
- `mcp-exposes-a-second-entry-point` — The MCP server is launched by the separate `reasons-mcp` console script rather than embedded in the `reasons` CLI.
