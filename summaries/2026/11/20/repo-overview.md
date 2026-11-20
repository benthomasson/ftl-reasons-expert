---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Repo Overview

# Repository Overview: `ftl-reasons` (v0.58.0)

A zero-dependency-core Python implementation of Doyle's (1979) Truth Maintenance System, packaged both as an importable library and as a CLI + MCP service that LLM agents can drive.

---

## 1. Purpose

The project implements a **Truth Maintenance System (TMS)** — a belief network where every assertion is a `node` (IN = believed, OUT = retracted), justified by `SL`/`CP` justifications, and linked by dependencies. It provides:

- **Retraction cascades**: retracting a premise automatically retracts anything that depends on it.
- **Restoration without rederivation**: re-asserting a premise recomputes dependents from the stored justification graph.
- **Dependency-directed backtracking (DDB)**: nogoods trigger backward traversal to the *minimal* responsible premise rather than arbitrary retraction.
- **Non-monotonic / dialectical reasoning**: `outlist` (`--unless`) supports default reasoning; `challenge`/`defend` implement formal argumentation.

The problem it solves: giving autonomous agents (and humans) a durable, inspectable, *retractable* store of beliefs with provenance, so that when a source changes or a contradiction surfaces, the entire downstream belief set degrades gracefully instead of silently contradicting itself.

---

## 2. Architecture

The design is a **layered, hexagonal-ish architecture** with a graph kernel at the center and many thin adapters around it.

```
                         ┌─────────────────────────┐
   cli.py ──────┐        │  Graph Kernel           │
   api.py ──────┤        │  network.py             │
   mcp_server.py┤──────▶ │  derive.py              │
   ask.py ──────┤        │  contradictions.py      │
                         │  (nodes, justifications)│
                         └───────────┬─────────────┘
                                     │
                         ┌───────────▼─────────────┐
                         │  Storage Layer          │
                         │  storage.py (SQLite)    │
                         │  pg.py     (Postgres)   │
                         │  metadata.py, verify.py │
                         └───────────┬─────────────┘
                                     │
                         ┌───────────▼─────────────┐
                         │  Event / Audit          │
                         │  pubsub.py              │
                         │  (propagation log)      │
                         └───────────┬─────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
     Import/Export             LLM Review/Repair              Integrity
     import_beliefs.py         llm.py, ask.py                merkle.py
     import_agent.py           review*.py, repair*.py        check_stale.py
     export_markdown.py        report*.py, propose_update.py build_wiki.py
     export_card.py            mind_service.py
     hf.py
```

Key design patterns:

- **Command dispatcher** in `cli.py` fans out to one module per command.
- **Storage abstraction** — a single logical schema (`storage.py`) with an optional PostgreSQL backend (`pg.py`), selected at runtime (see `test_pg_dispatch.py`).
- **Event bus (`pubsub.py`)** decouples state mutations from observers (the `reasons log` audit trail, cache invalidation, etc.).
- **Content addressing (`merkle.py`)** — belief integrity is verifiable without trusting the store.
- **Optional extras gate optional subsystems** — the core is pure stdlib; heavy deps load only when their feature is used.

---

## 3. Key Components

### Graph kernel
- **`network.py`** — the propagation engine: IN/OUT recomputation, cascade, restoration.
- **`derive.py`** — computes node truth values from justification validity.
- **`contradictions.py`** — nogood detection and DDB (dependency-directed backtracking).
- **`check_stale.py`** — compares recorded source hashes against the filesystem.
- **`compact.py`** — produces a token-budgeted textual summary for LLM context injection.

### Storage & integrity
- **`storage.py`** — SQLite schema and CRUD for nodes, justifications, nogoods, propagation log.
- **`pg.py`** — PostgreSQL backend (opt-in via `pg` extra).
- **`metadata.py`** — node metadata (labels, sources, source hashes, access tags).
- **`verify.py`** — consistency checker for the justification graph.
- **`merkle.py`** *(present only in `reasons/`, not `reasons_lib/`)* — hash-tree verification of the belief set.

### Interfaces
- **`cli.py`** → `reasons` executable. This is the primary user-facing surface (~30 subcommands).
- **`api.py`** — programmatic Python API (used by tests such as `test_api.py`).
- **`mcp_server.py`** → `reasons-mcp` executable. Exposes the belief store as MCP tools to Claude/other agents.
- **`mcp_client.py`** — client side, so `reasons` can call other MCP servers.

### LLM / agent workflows (opt-in)
- **`llm.py`** — provider abstraction over Anthropic / Vertex / OpenAI / Langfuse.
- **`ask.py`** — natural-language Q&A grounded in the belief graph.
- **`review.py`, `review_justifications.py`, `review_premises.py`** — LLM-driven audits of the graph.
- **`repair.py`, `repair_premises.py`** — propose and apply fixes to inconsistent justifications.
- **`report_belief.py`, `report_gated.py`** — generate belief reports (the "gated" variant likely requires a confidence threshold).
- **`propose_update.py`** — emits candidate graph edits; the `.forge/proposed-entries.json` + `.forge/report.json` pair are the artifacts of a self-update loop.
- **`mind_service.py`** — a higher-level long-lived service wrapping the TMS for agent memory.

### Import/Export & federation
- **`import_beliefs.py`** — reads a `beliefs.md` registry (auto-detects `nogoods.md`).
- **`import_agent.py`** — imports from agent trace/state formats.
- **`export_markdown.py`** — round-trippable `beliefs.md`.
- **`export_card.py`** — compact, portable summary card.
- **`build_wiki.py`** — renders the belief graph as a wiki-style site.
- **`hf.py`** — HuggingFace dataset export/import (see `test_import_hf.py`, `test_hf_default_org.py`).

### Clustering (opt-in via `cluster` extra)
- **`cluster.py`** — semantic clustering of beliefs using `sentence-transformers` and `scikit-learn`.

---

## 4. Data Flow

**Typical mutation flow** (`reasons retract <id>`):

1. `cli.py` parses argv and dispatches to `network.retract()`.
2. `storage.py` loads the node and its transitive dependents from SQLite/Postgres.
3. `derive.py` recomputes each dependent's IN/OUT status from justification validity.
4. Status changes are written back via `storage.py`.
5. `pubsub.py` publishes change events; the propagation log (`reasons log`) is updated.
6. `merkle.py` re-hashes affected subtrees if integrity tracking is enabled.

**Typical read flow** (`reasons explain`, `reasons trace`, `reasons compact`):

1. CLI dispatches to the appropriate module.
2. Module walks the justification graph backward (for `trace`) or forward (for `explain`).
3. Output is rendered as text/JSON/markdown.

**Import flow** (`reasons import-beliefs FILE`):

`import_beliefs.py` → parse markdown → `metadata.py` hashes referenced sources → `storage.py` bulk insert → `derive.py` initial propagation → `pubsub.py` emits events.

**LLM review flow**:

`review*.py` → `storage.py` reads → `llm.py` (provider-agnostic call) → optional `propose_update.py` → `.forge/proposed-entries.json` + `.forge/report.json` → human or automated approval → mutations written back.

---

## 5. Dependencies

`pyproject.toml` declares **`dependencies = []`** — the core parser, storage engine, and propagation logic are pure stdlib.

All non-core deps are gated behind extras:

| Extra | Deps | Why |
|---|---|---|
| `test` | pytest, pytest-cov, pyyaml | Test harness |
| `pg` / `test-pg` | `psycopg[binary]>=3.1` | Postgres backend |
| `cluster` | `sentence-transformers`, `scikit-learn` | Embedding-based clustering in `cluster.py` |
| `mcp` | `mcp>=1.0` | MCP server/client (`mcp_server.py`, `mcp_client.py`) |
| `api` | `langchain-anthropic` | Anthropic chat models |
| `vertex` | `langchain-google-vertexai` | Vertex AI |
| `openai` | `openai>=1.0` | OpenAI models |
| `langfuse` | `langfuse>=2.0` | LLM observability (see `costs/`) |
| `llm` | aggregate of the above | Convenience bundle |

The rationale is clear: the TMS *kernel* has no third-party footprint, so it can be embedded anywhere. Everything else is a plug-in.

---

## 6. Entry Points

Defined in `[project.scripts]`:

- **`reasons`** → `reasons.cli:main` — the CLI (also usable via `uvx ftl-reasons <command>`).
- **`reasons-mcp`** → `reasons.mcp_server:main` — the MCP service client-facing server.

Programmatic use is via `reasons/api.py`. Both entry points and `api.py` funnel through the same storage/graph stack.

---

## 7. Configuration

Configuration is **minimal and filesystem-driven**:

- **Storage location**: `reasons.db` in the working directory by default (`reasons init` creates it). Postgres connection strings are passed via CLI/env when `pg` extra is active.
- **`pyproject.toml`** — the only build-config surface (Python ≥3.10, package list `["reasons", "reasons_lib"]`, pytest testpath).
- **Optional dirs**: `.forge/` (proposed updates + report), `costs/` (LLM spend ledgers), `.claude/skills/reasons/SKILL.md` (a Claude skill manifest describing how the agent should invoke the CLI).
- **`CLAUDE.md`** — repo-specific guidance for LLM coding assistants.
- **No runtime config file** — behavior is driven by CLI flags (`--budget`, `--no-truncate`, `--unless`, `--sl`, `--cp`, `--force`, `--status`, `--premises`, etc.).

---

## 8. Module Boundaries

Observed import rules:

- **`storage.py` / `pg.py`** are the *only* modules that know about SQL. Everything else imports from `storage.py`'s façade.
- **`network.py`** is the *only* module that mutates IN/OUT truth values as part of a cascade. Other modules call into it.
- **`derive.py`** is called *by* `network.py`; it does not call back.
- **`llm.py`** is a leaf adapter — the `review*`, `repair*`, `ask`, and `report*` modules depend on it but it depends on none of them.
- **`cli.py` / `mcp_server.py` / `api.py`** are *adapters at the edge* — they import from any subsystem but nothing imports from them.
- **`reasons_lib/`** mirrors `reasons/` but is a **stale duplicate**: identical module set *except* `merkle.py`, which exists only in `reasons/`. Both are shipped (`packages = ["reasons", "reasons_lib"]`), yet only `reasons.*` has console scripts. Treat `reasons_lib/` as legacy/back-compat; new development belongs in `reasons/`. This is the biggest maintenance hazard in the repo — divergence between the two will silently accumulate.

---

## 9. Extension Points

| Goal | Where to extend |
|---|---|
| New CLI subcommand | Add handler in `reasons/cli.py`, delegate to a new or existing module |
| New justification type (beyond SL/CP) | `storage.py` schema + `derive.py` validity check + `network.py` propagation |
| New storage backend | Mirror `pg.py`; rely on the `pg_dispatch` selection pattern from `test_pg_dispatch.py` |
| New import/export format | Mirror `export_markdown.py` / `import_beliefs.py` |
| New LLM provider | Add a branch in `llm.py`; add an extra to `pyproject.toml` |
| New MCP tool | Register in `mcp_server.py` |
| New automated maintenance flow | Mirror `review_premises.py`; emit into `.forge/proposed-entries.json` for review |
| New integrity check | Add to `verify.py`; hook any new source-hash logic into `check_stale.py` |

---

## 10. Known Constraints

- **Python ≥ 3.10** (uses modern type syntax; cannot run on 3.9).
- **Single-writer SQLite by default** — concurrent writers will contend on the database lock. For multi-agent or multi-process deployments, the `pg` extra is required.
- **Opt-in subsystems**: MCP, LLM, clustering, and Postgres all require the corresponding extra to be installed; attempting to use them without the extra will fail at import time (this is a *feature* — it keeps the core dependency-free — but it means error messages surface at first use, not at install).
- **Source-hash staleness relies on filesystem access** — `check_stale.py` must be able to stat the paths recorded in `metadata.py`; on sandboxed/read-only deployments `check-stale` will report spurious staleness.
- **Two parallel packages** (`reasons/`, `reasons_lib/`) are shipped; consumers must pick one, and `merkle.py`'s absence from `reasons_lib/` means integrity verification is only available via `reasons.`.
- **CI**: `.github/workflows/tests.yml` runs `pytest` over `tests/` (211 tests per README, ~66 files).

---

## Topics to Explore

- [file] `reasons/network.py` — The propagation kernel: how cascade and restoration actually walk the dependency graph.
- [file] `reasons/storage.py` — Schema and façade for nodes/justifications/nogoods; the only SQL-aware module.
- [function] `reasons/cli.py:main` — The single dispatcher; the fastest map of every supported command.
- [diff] `reasons/` vs `reasons_lib/` — Diff the two trees to understand the divergence (currently only `merkle.py`) and which is authoritative for your use.
- [general] `forge-propose-update-loop` — How `propose_update.py` writes `.forge/proposed-entries.json` and `.forge/report.json`, and whether the loop is human-gated.

## Beliefs

- `pyproject-declares-no-required-runtime-dependencies` — `dependencies = []`; every non-stdlib package is behind an optional extra.
- `cli-entry-point-is-reasons-cli-main` — `[project.scripts] reasons = "reasons.cli:main"` is the sole CLI wiring.
- `mcp-entry-point-is-reasons-mcp-server-main` — `[project.scripts] reasons-mcp = "reasons.mcp_server:main"` is the sole MCP wiring.
- `reasons-lib-mirrors-reasons-except-merkle` — `reasons_lib/` contains every module in `reasons/` except `merkle.py`, which exists only in `reasons/`.
- `storage-supports-sqlite-by-default-and-postgres-optionally` — `storage.py` is stdlib SQLite; `pg.py` is gated behind the `pg` extra (psycopg).
- `justifications-support-outlist-for-non-monotonic-reasoning` — README documents `--unless` as an outlist enabling default reasoning; this is the basis for the `challenge`/`defend` commands.
