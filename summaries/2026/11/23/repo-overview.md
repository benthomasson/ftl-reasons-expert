---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Repo Overview

# Reasons — Repository Overview

## Purpose

`ftl-reasons` (v0.58.0) is a Python implementation of **Doyle's (1979) Truth Maintenance System (TMS)**. It models beliefs as nodes in a dependency graph, tracks whether each node is `IN` (believed) or `OUT` (retracted), and automatically propagates changes:

- A node with a **justification** (SL/CP) is `IN` when its antecedents support it.
- Retracting a premise triggers a **retraction cascade** through dependents; re-asserting it triggers **restoration without rederivation**.
- **Nogoods** (contradiction sets) trigger **dependency-directed backtracking** that retracts the *responsible premise* with minimal disruption — not just any participating node.
- **Outlists** (`--unless`) allow non-monotonic/default reasoning and dialectical **challenge/defend** argumentation.
- **Sources** carry hashes so `check-stale` can flag beliefs whose supporting files have changed.

The system is usable as a CLI, an MCP server, and an LLM-integrable reasoning service.

## Architecture

The project is layered and thin at the top, with a stdlib-only core:

```
┌──────────────────────────────────────────────────────────────┐
│ Entry Points                                                 │
│   reasons.cli:main                 reasons.mcp_server:main   │
├──────────────────────────────────────────────────────────────┤
│ Application Layer                                            │
│   cli.py, mcp_server.py, mcp_client.py, api.py               │
│   ask.py, propose_update.py, review*.py, repair*.py          │
│   import_*.py, export_*.py, build_wiki.py                    │
├──────────────────────────────────────────────────────────────┤
│ Domain Logic (reasoning)                                     │
│   derive.py, contradictions.py, check_stale.py,              │
│   verify.py, merkle.py, cluster.py, compact.py               │
├──────────────────────────────────────────────────────────────┤
│ Persistence & Infra                                          │
│   storage.py (SQLite default), pg.py (PostgreSQL optional)   │
│   network.py, pubsub.py, metadata.py, mind_service.py        │
├──────────────────────────────────────────────────────────────┤
│ Optional Integration                                         │
│   llm.py, hf.py, cluster.py (sentence-transformers),         │
│   openai/langchain/vertex extras                             │
└──────────────────────────────────────────────────────────────┘
```

Key design patterns:

- **Layered dispatch**: CLI parses arguments → resolves operations → calls into specialized modules → writes through `storage.py`.
- **Pluggable persistence**: `storage.py` provides the SQLite backend; `pg.py` provides a PostgreSQL backend reachable via the optional `pg` extra.
- **Adapter pattern** for I/O: `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `build_wiki.py` all translate between external formats and the internal node graph.
- **Gate/review pipeline**: `propose_update.py` + `review*.py` + `.forge/proposed-entries.json` implement a stage-then-apply workflow for externally generated belief changes — this is how an LLM or script proposes a mutation that is reviewed before it lands.
- **Optional dependency extras** keep the core TMS engine stdlib-only, so the CLI and priority logic work without installing an LLM SDK or a database driver.

## Key Components

| Module | Responsibility |
|--------|----------------|
| `reasons/cli.py` | Argument parser and command dispatch (`reasons = "reasons.cli:main"`) |
| `reasons/storage.py` | Node/justification/nogood persistence, SQLite schema (creates `reasons.db`) |
| `reasons/pg.py` | Optional PostgreSQL backend (`psycopg[binary]`) |
| `reasons/derive.py` | Backward chaining / derivation expansion with a token-budget guard (`test_derive_budget.py`) |
| `reasons/contradictions.py` | Nogood detection and dependency-directed backtracking |
| `reasons/check_stale.py` | Source-hash staleness check against `source` metadata |
| `reasons/merkle.py` | Merkle-tree verification of the belief graph (only present in `reasons/`, not `reasons_lib/`) |
| `reasons/review.py` / `review_justifications.py` / `review_premises.py` | Audit pipeline for reviewing proposed changes |
| `reasons/repair.py` / `repair_premises.py` | Automated repair of broken justifications (e.g. after retraction) |
| `reasons/import_beliefs.py` / `import_agent.py` | Import external registries (`beliefs.md` + `nogoods.md`) and agent-generated proposals |
| `reasons/import_agent.py` / `propose_update.py` | Agent-driven mutation staging (see `.forge/`) |
| `reasons/export_markdown.py` / `export_card.py` / `build_wiki.py` | Export beliefs as markdown/wiki/cards |
| `reasons/cluster.py` | Sentence-transformer clustering (`cluster` extra: `sentence-transformers`, `scikit-learn`) |
| `reasons/compact.py` | Token-budgeted belief summary for context injection |
| `reasons/llm.py` / `api.py` / `ask.py` | LLM-backed query/answer layer (extras: `openai`, `llm`, `vertex`, `langfuse`) |
| `reasons/hf.py` | HuggingFace dataset importer |
| `reasons/mcp_server.py` / `mcp_client.py` | Model Context Protocol server (`reasons-mcp`) and client |
| `reasons/network.py` / `pubsub.py` | Network abstraction and pub/sub for cross-process propagation |
| `reasons/metadata.py` | Node metadata (source URIs, labels, hashes) |
| `reasons/mind_service.py` | Higher-level "mind" service over the TMS (report/report-gated workflows) |

The `reasons_lib/` directory mirrors `reasons/` module-by-module **except** it lacks `merkle.py`. Given the shared packaging in `pyproject.toml` (`packages = ["reasons", "reasons_lib"]`) and the entry point targeting `reasons.cli`, this appears to be a legacy or vendored copy of the package. Verifying with a diff is a good first step for any developer touching both trees.

## Data Flow

### Typical CLI mutation (`reasons add …`)

```
argv
  → reasons.cli:main (parse)
  → storage.Storage (open/create reasons.db)
  → build node + justifications (inlist / outlist)
  → propagate() to recompute IN/OUT
  → cascade: any dependent whose justifications became invalid is set OUT
  → persist, print changes
```

### `nogood A B`

```
CLI → contradictions.py
  → record nogood row
  → dependency-directed backtracking:
       walk justification graph to identify the minimal premise(s) responsible
  → retract those premises
  → propagate
```

### `check-stale`

```
storage (maybe SQLite, maybe PG)
  → for each IN node with a `source` field:
       re-hash file, compare with stored hash
  → report fresh/stale counts
```

### Import (`import-beliefs FILE`)

```
markdown parser → node records + nogoods.md auto-detected sibling
  → storage bulk insert
  → propagate to reach fixpoint
  → report import counters
```

### Propose/review (`.forge/`)

```
propose_update.py → write proposal to .forge/proposed-entries.json
review*.py       → validate/gate proposals against current graph
                 → apply accepted entries, write .forge/report.json
```

## Dependencies

Core has **no runtime dependencies** (empty `[project] dependencies`). All integrations are optional extras:

| Extra | Purpose |
|-------|---------|
| `test` | `pytest`, `pytest-cov`, `pyyaml` |
| `pg` | `psycopg[binary]` for PostgreSQL backend (`pg.py`) |
| `cluster` | `sentence-transformers`, `scikit-learn` for semantic clustering (`cluster.py`) |
| `mcp` | `mcp` SDK for `reasons-mcp` server/client |
| `api` | `langchain-anthropic` for Anthropic LLM integration (`api.py`) |
| `vertex` | `langchain-google-vertexai` for Vertex AI |
| `openai` | `openai` SDK |
| `langfuse` | `langfuse` for LLM observability |
| `llm` | Convenience bundle combining the above |

This design lets the TMS engine run in a minimal environment (e.g. embedded in CI or a script) while enabling LLM/MCP/PG/clustering features when needed.

## Entry Points

- `reasons` → `reasons.cli:main` — the primary CLI (`init`, `add`, `retract`, `nogood`, `explain`, `challenge`/`defend`, `compact`, `check-stale`, `export-markdown`, etc.).
- `reasons-mcp` → `reasons.mcp_server:main` — exposes the TMS as an MCP server so an agent can query/mutate the belief store.

## Configuration

- **`pyproject.toml`** — package metadata, Python `>=3.10`, extras, entry points.
- **`reasons.db`** — default SQLite database (created by `reasons init`); its path is the working-directory default unless overridden by CLI flags.
- **Postgres** — enabled by the `pg` extra and a connection string (used by `pg.py`).
- **LLM** — configured via the relevant extra (e.g. `api` for Anthropic, `openai` for OpenAI) and API keys supplied via the environment.
- **`.forge/proposed-entries.json` + `.forge/report.json`** — the staging and reporting store for the propose/review pipeline.
- **`.claude/skills/reasons/SKILL.md`** — a Claude Code skill definition describing how an agent should invoke the `reasons` CLI.
- **`.github/workflows/tests.yml`** — CI runs the test suite.

## Module Boundaries

No enforced import-linter exists, but the dependency directions implied by the tree and extras are:

- `storage.py` is the low-level foundation; almost every module imports from it.
- `pg.py` mirrors `storage.py`'s interface for the alternate backend — callers should treat them through a shared storage interface rather than importing `pg` directly.
- `cli.py` is allowed to import from nearly anything; domain modules (e.g. `derive.py`, `contradictions.py`) should import storage and each other but not vice versa.
- `mcp_server.py` and `api.py` sit at the outer boundary and depend on domain modules, not the reverse.
- Optional integration modules (`llm.py`, `hf.py`, `cluster.py`, `openai`/`vertex`) must not be imported at module-load time by the core path — otherwise the “no runtime dependencies” promise breaks. They are imported lazily inside functions or behind feature flags.
- `reasons_lib/` and `reasons/` should not import each other. Whichever is canonical (likely `reasons/`, given the entry points) should be the only one extended going forward.

## Extension Points

- **New CLI commands**: add a parser branch and handler in `reasons/cli.py`, then implement a module alongside `derive.py`/`check_stale.py`.
- **New persistence backend**: implement a `Store` against the same interface that `storage.py` and `pg.py` provide, and gate it behind a new extra.
- **New import/export formats**: add a sibling to `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, or `export_card.py`; the internal graph representation is the stable contract.
- **New reasoning strategies**: hooks naturally fit in `derive.py` (derivation), `contradictions.py` (nogoods/backtracking), `repair.py` / `repair_premises.py` (recovery of broken justifications), and `review*.py` (audit logic).
- **New LLM providers**: `llm.py` / `api.py` are the seams for a new backend (note the existing `api`, `vertex`, `openai`, `langfuse` extras).
- **New MCP tools**: add tool registrations in `mcp_server.py`; keep them thin, delegating to domain modules and `storage`.
- **New propose/review pipeline stages**: extend `propose_update.py` plus the `review*.py` module family, writing results to `.forge/`.

## Known Constraints

- **Single-process default persistence**: SQLite is the default backend; concurrent multi-process writers on the same `.db` file are constrained by SQLite semantics. `pg.py` exists precisely to lift this when the `pg` extra is installed.
- **Python `>=3.10`** is required (`pyproject.toml`), so union types and structural pattern matching are available.
- **Stdlib-only core**: any code that eagerly imports an optional package (LLM SDK, psycopg, sentence-transformers, mcp) at module load will break the default install.
- **Staleness checks require filesystem access**: `check-stale` re-hashes source files, so it is meaningful only where the referenced paths exist locally.
- **`reasons` vs `reasons_lib` duplication** is a maintenance hazard: the two packages must be kept in sync by hand unless one is refactored to re-export from the other. Any change to shared modules should be mirrored or consolidated.
- **Test suite is large and semantic**: 70+ test files (211 tests per README) cover propagation, backtracking, non-monotonic semantics, dialectics, and round-trips. Changes to `storage.py` or the propagation engine should not be merged without running the full suite (`uv run --extra test pytest tests/ -v`).

## Topics to Explore

- [file] `reasons/storage.py` — The persistence layer; reading it first clarifies the node/justification/nogood schema that every other module builds on.
- [function] `reasons/cli.py:main` — The single dispatch point for every `reasons` subcommand; the most efficient way to see how user intent maps to domain modules.
- [file] `reasons/contradictions.py` — Contains the dependency-directed backtracking logic that is the system's distinctive capability over plain TMS.
- [general] `reasons-versus-reasons-lib-divergence` — Diff the two packages to determine which is canonical and whether `reasons_lib` should be deleted, vendored, or made a thin re-export.
- [general] `forge-propose-review-pipeline` — How `.forge/proposed-entries.json` is produced by `propose_update.py` and consumed by `review*.py`; this is the seam for agent-driven belief mutation.

## Beliefs

- `core-has-no-required-runtime-dependencies` — `[project] dependencies` in `pyproject.toml` is empty; every external integration (PG, LLM, MCP, clustering) is an optional extra, so the core TMS runs on stdlib alone.
- `cli-and-mcp-are-separate-console-scripts` — The package exposes exactly two console scripts: `reasons` → `reasons.cli:main` and `reasons-mcp` → `reasons.mcp_server:main`.
- `reasons-lib-lacks-merkle-module` — `reasons_lib/` contains every module in `reasons/` except `merkle.py`; the shared layout mirrors 34 modules with that one divergence.
- `sqlite-is-default-pg-is-optional` — Persistence defaults to SQLite via `storage.py` (creates `reasons.db` with `reasons init`); PostgreSQL is available only through the `pg` extra which installs `psycopg[binary]` for `pg.py`.
- `outlist-enables-non-monotonic-inference` — Justifications accept an outlist (`--unless Y`) that is valid only when listed antecedents are `OUT`, enabling default reasoning and the `challenge`/`defend` dialectical commands documented in the README.
