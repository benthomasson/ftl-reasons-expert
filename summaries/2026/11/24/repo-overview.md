---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Repo Overview

# Repository Overview: `ftl-reasons`

## 1. Purpose

`ftl-reasons` is a Python implementation of **Doyle's (1979) Truth Maintenance System (TMS)** — a dependency-network reasoning engine for tracking beliefs with automatic retraction cascades and restoration. It is packaged as both a CLI (`reasons`) and an MCP server (`reasons-mcp`), with SQLite and PostgreSQL persistence backends.

The core problem it solves: *"which beliefs must I retract (or restore) when evidence changes?"* This is the classic TMS problem — when a premise is retracted, every belief that only depended on it must go OUT too, and dependency-directed backtracking must find the *responsible premise* (not an arbitrary node) whenever a contradiction (nogood) is detected. The project extends the classic TMS with non-monotonic reasoning (outlist), dialectical argumentation (challenge/defend), Merkle-style source hashing, and pipeline-style LLM/agent integrations.

---

## 2. Architecture

### Package Mirroring (Important)

The repo ships **two parallel Python packages** that share nearly identical module names:

- `reasons/` — 34 Python files
- `reasons_lib/` — 33 Python files

The **only structural difference** is that `reasons/merkle.py` exists and `reasons_lib/merkle.py` does not. Both are declared in `[tool.setuptools] packages`. This strongly suggests `reasons_lib` is a *compatibility mirror / legacy namespace* kept in sync with `reasons` (with `merkle` being a newer addition that hasn't been ported). The name `reasons_lib` is the "library" surface; `reasons` is the canonical package and the target of both console script entry points.

### Layering (inferred from module names)

| Layer | Modules |
|---|---|
| **Frontends** | `cli.py`, `mcp_server.py`, `mcp_client.py` |
| **Service / API facade** | `api.py`, `mind_service.py`, `ask.py` |
| **Reasoning engine** | `derive.py`, `repair.py`, `repair_premises.py`, `contradictions.py`, `verify.py`, `check_stale.py` |
| **Storage** | `storage.py` (SQLite), `pg.py` (Postgres) |
| **Model / metadata** | `metadata.py`, `network.py`, `cluster.py`, `merkle.py` |
| **Presentation / export** | `export_markdown.py`, `export_card.py`, `build_wiki.py`, `compact.py` |
| **Agent workflows** | `propose_update.py`, `report_belief.py`, `report_gated.py`, `review*.py`, `import_agent.py` |
| **Interop** | `import_beliefs.py`, `hf.py`, `pubsub.py`, `llm.py` |

The **design pattern** is a layered *core-with-adapters*: the reasoning engine (`derive`, `repair`, `contradictions`, `verify`) is pure and I/O-free; storage and frontends are adapters; LLM and MCP are optional edge integrations gated behind extras.

---

## 3. Key Components

**Reasoning core**
- `reasons/derive.py` — Evaluates justifications (SL/CP) and produces IN/OUT truth values for nodes.
- `reasons/contradictions.py` — Manages **nogoods** (sets of nodes that can't all be IN).
- `reasons/repair.py` / `repair_premises.py` — **Dependency-directed backtracking**: walks the justification graph backward and retracts the minimal-disruption premise.
- `reasons/verify.py` — Integrity/consistency checks (`reasons propagate` runs the full recompute).
- `reasons/check_stale.py` — Compares source-file hashes against node metadata to surface stale IN nodes.

**Persistence**
- `reasons/storage.py` — SQLite backend (creates `reasons.db` via `reasons init`).
- `reasons/pg.py` — PostgreSQL backend, optional (`[pg]` extra, `psycopg[binary]`).

**CLI / API**
- `reasons/cli.py` — The `reasons` command. Contains subcommands `init`, `add`, `retract`, `assert`, `status`, `show`, `explain`, `challenge`, `defend`, `nogood`, `trace`, `hash-sources`, `propagate`, `log`, `search`, `list`, `import-beliefs`, `export`, `export-markdown`, `check-stale`, `compact`.
- `reasons/api.py` — Programmatic façade used by CLI, MCP, and services.
- `reasons/ask.py` — Query layer (search / lookup logic).

**Agent / MCP integration**
- `reasons/mcp_server.py` — Exposes TMS functions as MCP tools (`reasons-mcp` entry point).
- `reasons/mcp_client.py` — Client side of MCP.
- `reasons/mind_service.py` — Likely a high-level service orchestrating monitoring/reporting.
- `reasons/propose_update.py`, `report_belief.py`, `report_gated.py` — Agent-facing operations, including gating logic for entering proposed beliefs into the network.
- `reasons/review.py`, `review_justifications.py`, `review_premises.py` — Human/agent review workflows.
- `reasons/import_agent.py` — Ingests an agent's belief state.
- `reasons/pubsub.py` — Pub/sub for change events.

**Format interop**
- `reasons/import_beliefs.py` — Parses `beliefs.md` registries (and auto-detects adjacent `nogoods.md`).
- `reasons/export_markdown.py`, `export_card.py`, `build_wiki.py` — Output renderers.
- `reasons/compact.py` — Token-budgeted summary for context injection.
- `reasons/hf.py` — Hugging Face Hub interaction (model/artifact exchange).
- `reasons/merkle.py` — Content hashing for source integrity (only in `reasons`, not `reasons_lib`).

**LLM / semantic**
- `reasons/llm.py` — Chat model abstraction (LangChain-family providers).
- `reasons/network.py` — Graph/network utilities.
- `reasons/cluster.py` — Semantic clustering (sentence-transformers + scikit-learn).
- `reasons/metadata.py` — Node metadata schema.

---

## 4. Data Flow

```
User/Agent
   │
   ├─(CLI: reasons ...)──────────────┐
   ├─(MCP: reasons-mcp JSON-RPC)─────┤
   └─(Python: reasons.api.*)─────────┘
                                     │
                              ┌──────▼───────┐
                              │  cli.py /    │
                              │  mcp_server  │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │   api.py     │◄── ask.py (search)
                              │  (service)   │
                              └──────┬───────┘
                                     │
              ┌──────────────────────┼─────────────────────┐
              │                      │                     │
      ┌───────▼──────┐        ┌──────▼──────┐       ┌──────▼───────┐
      │  derive.py   │        │  storage.py │       │  metadata.py │
      │ contradictions│       │   / pg.py   │       │  merkle.py   │
      │   repair.py  │        │  (persist)  │       │  network.py  │
      │   verify.py  │        └─────────────┘       └──────────────┘
      └───────┬──────┘
              │
        ┌─────▼──────┐
        │  audit log │  (reasons log)
        └────────────┘
```

**Concrete flows**

1. **Mutation (assert / retract / add)** — CLI parses args → `api` → `storage` writes → `derive` recomputes truth values → `verify` checks → cascade updates propagate through dependents → `log` records entries.
2. **Contradiction** — `nogood` command → `contradictions` stores set → `repair`/`repair_premises` trace justification graph and retract responsible premise(s).
3. **Import** — `import_beliefs.py` parses `beliefs.md` (and auto-detected `nogoods.md`) → nodes + justifications + nogoods created → full propagation pass.
4. **Export** — read nodes from `storage` → `export_markdown.py` / `export_card.py` / `build_wiki.py` → markdown/JSON output; `compact.py` produces a token-budgeted variant.
5. **Staleness** — `check_stale.py` reads source hashes stored in `metadata` (via `merkle`) and compares against disk.
6. **Agent flow** — `report_belief.py` / `report_gated.py` submit claims → `propose_update.py` builds an update → `review*.py` validates → persisted through `api`/`storage`.

---

## 5. Dependencies

The **base install has zero runtime dependencies** (`dependencies = []`) — the TMS engine is pure Python. Everything else is opt-in via extras:

| Extra | Packages | Why |
|---|---|---|
| `test` | `pytest`, `pytest-cov`, `pyyaml` | Test tooling and fixture loading |
| `pg` | `psycopg[binary]>=3.1` | PostgreSQL backend (`pg.py`) |
| `cluster` | `sentence-transformers>=2.2`, `scikit-learn>=1.2` | Semantic clustering (`cluster.py`) |
| `mcp` | `mcp>=1.0` | Model Context Protocol server (`mcp_server.py`) |
| `api` | `langchain-anthropic>=0.3` | Anthropic chat models (`llm.py`) |
| `vertex` | `langchain-google-vertexai>=2.0` | Google Vertex AI models |
| `langfuse` | `langfuse>=2.0` | LLM observability / cost tracking (ties to `costs/`) |
| `openai` | `openai>=1.0` | OpenAI models |
| `llm` | anthopic + vertex + langfuse | Convenience umbrella |

The zero-base-dependency decision keeps `ftl-reasons` usable as a minimal library — you can import `reasons` and run the TMS engine without pulling in any third-party SDK.

---

## 6. Entry Points

From `pyproject.toml`:

```toml
[project.scripts]
reasons     = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

- `reasons <subcommand> [...]` — CLI (all user-facing operations).
- `reasons-mcp` — MCP server, speaks JSON-RPC over stdio for tool-based agent access.
- Python: `from reasons import api` and use the service façade directly.

Test entry via pytest: `uv run --extra test pytest tests/ -v` (211 tests, per README).

---

## 7. Configuration

- **Python**: `requires-python = ">=3.10"`.
- **Build**: `setuptools>=68.0` via `setuptools.build_meta`.
- **Packaging**: both `reasons` and `reasons_lib` are installed as packages.
- **Tests**: `[tool.pytest.ini_options] testpaths = ["tests"]`.
- **Runtime DB**: SQLite file `reasons.db` created by `reasons init`; Postgres via the `[pg]` extra (connection presumably configured via `DATABASE_URL`-style env in `pg.py` — not visible in the manifest).
- **LLM**: provider chosen via extras and initialized in `llm.py`; API keys are consumed from the standard provider env vars (Anthropic, Vertex, OpenAI).
- **CI**: `.github/workflows/tests.yml` — runs the pytest suite.
- **Observability**: `costs/` holds Langfuse-style cost dumps (e.g., `costs/2026-11-19T180331-update.json`); `.forge/` holds structured change records (`proposed-entries.json`, `report.json`); `.claude/skills/reasons/SKILL.md` provides agent-facing skill docs.

---

## 8. Module Boundaries

Enforced (or implied) rules:

1. **Reasoning core has no I/O** — `derive.py`, `contradictions.py`, `repair.py`, `repair_premises.py`, `verify.py` operate on in-memory node/justification state. All side effects live in `storage.py` / `pg.py` / `api.py`.
2. **Frontends are thin** — `cli.py` and `mcp_server.py` call `api.py`; business logic does not sit in the CLI.
3. **Storage is backend-pluggable** — `storage.py` (SQLite) and `pg.py` (Postgres) mirror each other's API, so `api.py` treats them interchangeably.
4. **Mirroring invariant** — `reasons/` and `reasons_lib/` must stay behaviorally identical. Any new module in `reasons/` requires a counterpart in `reasons_lib/` (currently `merkle.py` is the *only* exception, meaning the mirror is out of sync).
5. **Optional integrations live at the edge** — `llm.py`, `cluster.py`, `hf.py`, `mcp_client.py`, `mcp_server.py`, `pg.py` import packages only present when the matching extra is installed; the core must never import them unconditionally.
6. **Pub/sub is side-band** — `pubsub.py` lets external listeners observe changes without coupling the engine to a messaging layer.

---

## 9. Extension Points

Where a developer would add functionality:

| Goal | Where to go |
|---|---|
| **New CLI subcommand** | Add a command in `reasons/cli.py`, delegate to `reasons/api.py` |
| **New MCP tool** | Register in `reasons/mcp_server.py` |
| **New justification type (e.g., a new SL/CP variant)** | Implement in `reasons/derive.py` and validate in `reasons/verify.py` |
| **New consistency check** | Add to `reasons/verify.py` or `reasons/contradictions.py` |
| **New storage backend (e.g., DuckDB, S3)** | Mirror `storage.py`/`pg.py`; wire through `api.py` |
| **New import/export format** | Pattern after `import_beliefs.py`, `export_markdown.py`, `export_card.py` |
| **New LLM provider** | Add an extra in `pyproject.toml` and a branch in `reasons/llm.py` |
| **New agent workflow** | Pattern after `report_gated.py` → `propose_update.py` → `review*.py` |
| **New source-hash scheme** | Extend `reasons/merkle.py` (and port to `reasons_lib/merkle.py` if `reasons_lib` is kept) |

---

## 10. Known Constraints

- **SQLite single-writer** — The default backend (`reasons.db`) is a file; it's fine for single-user/CLI use but not safe for many concurrent writers. For concurrent deployment, use the `[pg]` extra and `pg.py`.
- **Python 3.10+** — `requires-python = ">=3.10"` means `match`/PEP 604 syntax is available; older interpreters are unsupported.
- **Optional-feature drift** — `reasons_lib/merkle.py` is missing, so the mirror is *currently inconsistent*. Any consumer of `reasons_lib.merkle` will fail.
- **LLM features require network + provider keys** — `llm.py`, `cluster.py`, `hf.py` call out to remote services; tests that exercise them (e.g., `test_import_hf.py`, `test_llm.py`) likely skip or mock when extras aren't installed.
- **Model downloads** — `cluster` pulls sentence-transformer weights on first run; first-call latency and disk usage should be expected.
- **Cost tracking is file-based** — `costs/*.json` appears to be an append-only Langfuse export rather than a live DB, so cost data is point-in-time.
- **No async contract** — The engine and storage are synchronous; concurrency safety depends on the storage backend.

---

## Topics to Explore

- [file] `reasons/derive.py` — The justification evaluator; this is where SL/CP truth-value semantics and the retraction cascade are actually computed.
- [file] `reasons/repair.py` — The dependency-directed backtracking algorithm; understanding this is required to reason about *which* premise gets retracted on a nogood.
- [file] `reasons/storage.py` — SQLite schema and round-trip behavior (`test_roundtrip.py`, `test_storage.py`); the source of truth for node/justification/log persistence.
- [file] `reasons/pg.py` — The PostgreSQL backend; check how it mirrors `storage.py` to understand the storage-pluggable contract.
- [general] `reasons-vs-reasons-lib-mirror` — How the two packages are kept in sync, why `merkle.py` is missing from `reasons_lib`, and whether one is slated for deprecation.
- [file] `reasons/cli.py` — Full map of subcommands and how each delegates to `api.py`.

---

## Beliefs

- `reasons-has-reasons-lib-mirror` — `reasons/` and `reasons_lib/` contain nearly identical module sets, with `merkle.py` present only in `reasons/`.
- `core-has-zero-runtime-dependencies` — `pyproject.toml` declares `dependencies = []`; all capabilities beyond the TMS engine ride on optional extras.
- `sqlite-is-the-default-backend` — `reasons init` creates `reasons.db`; PostgreSQL is only available via the `pg` extra using `psycopg[binary]>=3.1`.
- `mcp-server-is-a-separate-console-script` — `reasons-mcp` maps to `reasons.mcp_server:main`, distinct from the CLI entry `reasons.cli:main`.
- `outlist-enables-non-monotonic-reasoning` — Justifications accepting `--unless Y` encode default reasoning: the node is IN only while `Y` is OUT.
- `import-beliefs-auto-detects-nogoods-md` — `reasons import-beliefs FILE` auto-loads `nogoods.md` located next to `beliefs.md` unless `--nogoods` overrides.
