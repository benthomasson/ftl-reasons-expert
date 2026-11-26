---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Repo Overview

# Repository Overview: `ftl-reasons`

## Purpose

`ftl-reasons` is a Python implementation of **Doyle's (1979) Truth Maintenance System (TMS)**. It solves a specific reasoning problem: how to keep a large set of interdependent beliefs *consistent* and *auditable* as facts change.

Instead of manually updating everything when an assumption is invalidated, you register beliefs as nodes in a dependency network, and the system:

- Cascades **retraction** when a premise goes OUT
- Cascades **restoration** automatically when it comes back IN
- Detects **contradictions** (nogoods) and uses **dependency-directed backtracking** to identify which *premise* to retract with minimal disruption
- Supports **non-monotonic reasoning** via justifications with an "outlist" ("believe X unless Y")
- Persists an **audit trail** that supports `explain`, `trace`, and `log` commands

The README frames it as a belief database with a CLI, rather than a library — e.g. `reasons add`, `reasons retract`, `reasons explain`, `reasons check-stale`. The library form is importable as `reasons.*` (and mirrored in `reasons_lib.*`).

---

## Architecture

The system follows a **layered, storage-agnostic core with pluggable backends and optional AI integrations**:

```
┌────────────────────────────────────────────────────────────┐
│ Entry points:  reasons/cli.py      reasons/mcp_server.py   │
├────────────────────────────────────────────────────────────┤
│ Orchestration: mind_service.py · propose_update.py         │
│                review*.py · repair*.py · report*.py        │
├────────────────────────────────────────────────────────────┤
│ Truth maintenance core:                                    │
│   network.py (graph)   derive.py (SL/CP)   verify.py       │
│   contradictions.py    check_stale.py      merkle.py       │
├────────────────────────────────────────────────────────────┤
│ Persistence: storage.py (SQLite)  ·  pg.py (Postgres)      │
├────────────────────────────────────────────────────────────┤
│ Optional integrations:                                     │
│   llm.py / api.py · ask.py · cluster.py · hf.py ·          │
│   mcp_client.py · import_agent.py · pubsub.py              │
└────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

1. **Zero required dependencies.** The core TMS logic is pure Python (`dependencies = []`). Every integration — Postgres, LLM providers, MCP, embeddings — is an optional extra in `pyproject.toml`. This makes the belief engine trivially embeddable and testable.
2. **Dual package layout.** `reasons/` and `reasons_lib/` are near-identical parallel packages (both shipped by setuptools). `reasons_lib/` contains 33 of the 34 files in `reasons/` — it is missing only `merkle.py`. This is best treated as an inheritance/compatibility shim: `reasons` is the canonical package (used by the console scripts), while `reasons_lib` is a legacy/library-only surface that has drifted slightly out of sync.
3. **Storage abstraction with backend dispatch.** `storage.py` provides the default SQLite backend and `pg.py` provides Postgres; test names like `test_pg_dispatch.py` suggest a dispatcher chooses between them at runtime.
4. **Content-addressable node identity.** `merkle.py` computes deterministic hashes over node content/metadata. This powers staleness detection, deduplication (`test_deduplication.py`), and source-file freshness (`check_stale.py`, `test_check_stale_issue25.py`).
5. **Everything is auditable.** Propagation is a first-class operation (`reasons propagate`) that records to a log; `explain` walks backwards through the justification graph, and `trace` walks forward to identify assumption roots.

---

## Key Components

### Truth-maintenance core

| Module | Responsibility |
|--------|----------------|
| `reasons/network.py` | Node/edge graph of beliefs and justifications; the substrate for propagation and dependents lookup. |
| `reasons/derive.py` | SL and CP justification evaluation; supports budgeted derivation (`test_derive_budget.py`). |
| `reasons/verify.py` | Post-propagation invariant checks. |
| `reasons/contradictions.py` | Contradiction detection surfaced as nogoods. |
| `reasons/merkle.py` | Content hashing for node identity, deduplication, and integrity. |
| `reasons/check_stale.py` | Compares recorded source-file hashes against current filesystem state (`reasons check-stale`, `hash-sources`). |
| `reasons/repair.py` / `repair_premises.py` | Repair workflows when stale/inconsistent premises are discovered. |
| `reasons/review_premises.py` / `review_justifications.py` / `review.py` | Human-in-the-loop review commands, likely LLM-assisted. |

### Persistence

| Module | Responsibility |
|--------|----------------|
| `reasons/storage.py` | Default SQLite backend + import/export (`reasons export`, `reasons import-beliefs`). |
| `reasons/pg.py` | Postgres backend (`pg` extra). |
| `reasons/metadata.py` | Node metadata (sources, labels, hashes). |

### Interop / import / export

| Module | Responsibility |
|--------|----------------|
| `reasons/import_beliefs.py` | Parses a `beliefs.md` registry and auto-detects adjacent `nogoods.md`. |
| `reasons/import_agent.py` | Imports state produced by another agent. |
| `reasons/hf.py` | HuggingFace dataset round-trip. |
| `reasons/export_markdown.py` | Emits `beliefs.md`-compatible markdown. |
| `reasons/export_card.py` | Concise "card" export. |
| `reasons/build_wiki.py` | Generates a wiki-style site. |

### AI / reasoning helpers

| Module | Responsibility |
|--------|----------------|
| `reasons/llm.py` | Thin provider abstraction over Anthropic, Vertex, Langfuse, OpenAI extras. |
| `reasons/api.py` | `langchain-anthropic`-based API surface (`api` extra). |
| `reasons/ask.py` | Natural-language querying over the belief network. |
| `reasons/cluster.py` | Embedding-based clustering (`sentence-transformers` + `scikit-learn`). |
| `reasons/propose_update.py` | Generates proposed belief updates for review. |
| `reasons/report_belief.py` / `report_gated.py` | Gated reporting — likely a review gate before suggestions are written. |
| `reasons/compact.py` | Token-budgeted summary for context injection (`reasons compact --budget`). |

### Interfaces

| Module | Responsibility |
|--------|----------------|
| `reasons/cli.py` | Main CLI; dispatches every `reasons …` subcommand. |
| `reasons/mcp_server.py` | Model Context Protocol server (`reasons-mcp`). |
| `reasons/mcp_client.py` | MCP client used to call out to external servers (e.g. `test_ask_mcp.py`). |
| `reasons/mind_service.py` | Higher-level service facade wrapping storage + propagation + helpers. |
| `reasons/pubsub.py` | In-process publish/subscribe for propagation events. |

---

## Data Flow

**Typical mutation path** (`reasons add`, `retract`, `assert`, `nogood`, `propagate`):

```
CLI arg ──► cli.main ──► mind_service / storage
                             │
                             ▼
              load nodes + justifications + nogoods
                             │
                             ▼
                 network.py + derive.py evaluate SL/CP
                             │
                             ├─► outlist check (non-monotonic)
                             ├─► contradictions.py → new nogoods
                             └─► dependency-directed backtracking
                             │
                             ▼
              recompute IN/OUT; cascade retraction or restoration
                             │
                             ▼
              storage.py / pg.py persist; propagate log append
                             │
                             ▼
                  pubsub.publish(event) [if subscribed]
```

**Read paths:**

- `reasons status` / `list` / `show` — read storage, format output.
- `reasons explain` — walk backwards along valid justifications to premises.
- `reasons trace` — walk **forward** from a target to premise roots.
- `reasons log` — replay propagation audit.
- `reasons check-stale` — read node source metadata (hashes), stat the file, compare via `merkle`.

**Import/export:**

- `import-beliefs FILE` → `import_beliefs.py` → resolves relative `nogoods.md` → emits storage insertions → triggers propagation to reconcile contradictions.
- `export-markdown` / `export` / `export-card` / `build-wiki` → read storage → serializers.

**LLM-assisted paths** (require an `llm`/`api`/`vertex`/`openai` extra):

- `ask.py` → `llm.py` → provider; context comes from `compact.py` and/or search.
- `propose_update.py` → generates suggestions → `report_gated.py` / `review.py`.
- `cluster.py` embeds node text → groups semantically similar nodes; results can feed `deduplication` and `contradictions`.

---

## Dependencies

All runtime dependencies are **optional extras**, declared in `pyproject.toml`:

| Extra | Packages | Why |
|-------|----------|-----|
| `test` | pytest, pytest-cov, pyyaml | Test suite + YAML fixtures. |
| `pg` / `test-pg` | psycopg[binary]≥3.1 | Postgres backend for `pg.py`. |
| `cluster` | sentence-transformers≥2.2, scikit-learn≥1.2 | Semantic clustering in `cluster.py`. |
| `mcp` | mcp≥1.0 | MCP server/client. |
| `api` / `vertex` / `langfuse` / `openai` | langchain-anthropic, langchain-google-vertexai, langfuse, openai | LLM providers. |
| `llm` | Anthropic + Vertex + Langfuse | Aggregate for `llm.py` consumers. |

The empty core `dependencies` is a deliberate constraint: the TMS must be installable and testable without any AI/DB heavy lifting.

---

## Entry Points

Defined in `[project.scripts]`:

- `reasons = "reasons.cli:main"` — the main CLI.
- `reasons-mcp = "reasons.mcp_server:main"` — MCP server for agent integration.

Packaging is setuptools (`build-backend = "setuptools.build_meta"`), `requires-python >=3.10`, MIT license, version `0.58.0`. Both `reasons` and `reasons_lib` are declared as packages.

Typical invocations from the README: `reasons init`, `reasons add ID "text" --sl a,b`, `reasons retract ID`, `reasons nogood A B`, `reasons check-stale`, `reasons compact --budget 500`.

---

## Configuration

There is no dedicated config file — configuration happens through:

- **CLI flags** (`--sl`, `--cp`, `--unless`, `--budget`, `--status`, `--premises`, …) parsed in `cli.py`.
- **Environment variables** consumed by `llm.py` / `api.py` / `pg.py` (standard for LangChain / psycopg).
- **`pyproject.toml` extras** — the presence of a specific extra determines which plugins (Postgres, MCP, LLM) are available at runtime.
- **Per-node metadata** via `metadata.py` (source paths, labels, hashes) — this is *data* configuration, not system config.

The on-disk state is `reasons.db` (SQLite) unless Postgres is used. `.forge/` and `costs/` appear to be repo-local artifacts (see below).

---

## Module Boundaries

There is no enforced import-linter, but the code and tests imply these rules:

1. **Storage backends (`storage.py`, `pg.py`) must not import the AI/LLM modules.** This is what preserves the zero-dependency core and makes the core unit-testable without extras.
2. **The truth-maintenance core (`network.py`, `derive.py`, `verify.py`, `contradictions.py`, `merkle.py`, `check_stale.py`) must not import `cli.py` or `mcp_server.py`.** CLI concerns stay in `cli.py`.
3. **`cli.py` and `mcp_server.py` are the only modules that translate user input into operations.** Everything below them is callable programmatically.
4. **`mind_service.py` is the intended façade for higher-level consumers** (multiple entry points call into it, not directly into `storage.py`).
5. **`storage.py` and `pg.py` are peers, not layered** — one does not import the other; a dispatcher picks the backend (`test_pg_dispatch.py` is the tell).

`reasons/` and `reasons_lib/` violate any strict DRY rule — they are duplicates. The practical convention is: **`reasons_lib` is the library-only surface (no `merkle.py`) and `reasons` is canonical**. Any change to core logic should be made in both, or better, `reasons_lib` should be treated as deprecated and re-exported from `reasons`.

---

## Extension Points

- **New storage backend** — add a new peer module alongside `storage.py`/`pg.py` (e.g. `redis_storage.py`) and wire it into the backend dispatch (referenced by `test_pg_dispatch.py`).
- **New justification type** — extend `derive.py`; SL and CP are already handled, and the README references Dijkstra-style SL/CP/outlist.
- **New CLI subcommand** — add to `cli.py` and delegate to `reasons/<newmodule>.py` (mirroring `export_markdown`, `compact`, `check_stale`).
- **New MCP tool** — add to `mcp_server.py` (see `test_ask_mcp.py`).
- **New LLM provider** — extend `llm.py` and add an extra to `pyproject.toml`.
- **New export format** — peer to `export_markdown.py`, `export_card.py`, `build_wiki.py`.
- **New repair/review strategy** — peer to `repair.py` / `repair_premises.py` / `review_*.py`; these are already pluggable by name and exercised by matching tests.
- **Event consumers** — subscribe via `pubsub.py` (see `test_pubsub.py`).

---

## Known Constraints

- **Single-process / single-writer assumption.** SQLite + in-process `pubsub` imply the default deployment is one process. Postgres (`pg.py`) relaxes this somewhat, but the propagation model is still "recompute the whole dependent subgraph on mutation."
- **No async-first API.** Entry points and core are synchronous (`cli.main`, `mcp_server.main`); `asyncio` appears only where imposed by external SDKs (MCP, LLMs).
- **Source-staleness is filesystem-based.** `check_stale.py` stats referenced files; if a source path is missing or a node lacks a recorded hash, it is "fresh" by default until `hash-sources` is run. `test_check_stale_issue25.py` suggests at least one real-world gotcha has already been fixed.
- **`reasons.db` is authoritative local state.** Corruption or schema drift requires migration — `test_migrate_defeaters.py` indicates migrations exist for the "defeaters" feature.
- **Python ≥3.10** required; uses modern typing and likely structural pattern matching.
- **Optional extras are not lazy-imported everywhere.** `tests/conftest.py` likely guards imports; if a caller imports `reasons.ask` without the `llm` extra installed, it will fail at import time.
- **Repo artifacts `.forge/` (proposed-entries.json, report.json) and `costs/2026-11-19T180331-update.json`** are not part of the shipped package — they appear to be CI/agent-generated review artifacts and LLM cost logs, respectively. Treat them as scratch output, not code.
- **Duplicated package drift risk.** Any fix applied only to `reasons/` (e.g. a change to `merkle.py`) will not propagate to `reasons_lib/`.

---

## Topics to Explore

- [file] `reasons/storage.py` — The default persistence layer, backend dispatch, and JSON/markdown import/export primitives
- [file] `reasons/derive.py` — The SL/CP justification evaluator and its derivation-budget mechanism
- [file] `reasons/network.py` — The dependency-graph substrate that makes cascades and backtracking possible
- [function] `reasons/cli.py:main` — The command dispatcher; the map of the entire public CLI surface
- [general] `reasons-vs-reasons-lib-divergence` — Why two parallel packages ship, whether one is deprecated, and how to keep them in sync

---

## Beliefs

- `zero-required-runtime-dependencies` — `pyproject.toml` declares `dependencies = []`; all capabilities come through optional extras.
- `reasons-has-duplicate-reasons-lib` — Both `reasons/` and `reasons_lib/` are shipped as packages, and `reasons_lib/` is missing exactly `merkle.py` relative to `reasons/`.
- `storage-is-dual-backend` — Persistence is provided by `storage.py` (SQLite) and `pg.py` (Postgres), selected via a backend dispatcher (see `tests/test_pg_dispatch.py`).
- `merkle-hashing-enables-staleness-checks` — `merkle.py` provides the content hashing used by `check_stale.py` and `reasons hash-sources`.
- `cli-and-mcp-are-the-only-entry-points` — `[project.scripts]` exposes only `reasons.cli:main` and `reasons.mcp_server:main`; all other modules are called programmatically.
