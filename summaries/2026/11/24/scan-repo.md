---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Scan: repo

# Reconnaissance Report

**Quick observation before anything else:** `reasons/` and `reasons_lib/` are near-identical file trees (same ~34 modules), and `pyproject.toml` installs **both** as packages with identical script targets. The only obvious divergence is `reasons/merkle.py` (present in `reasons/`, absent from `reasons_lib/`). This is a real architectural smell to resolve early — either one is a stale/generated mirror, or there's a subtle difference in runtime behavior. Do not assume `reasons_lib` is a shim; verify against `CHANGELOG.md` and `CLAUDE.md`.

---

## 1. Critical Files

| # | File | Why it matters |
|---|------|----------------|
| 1 | `README.md` | Definitional ground truth for the TMS: **IN/OUT**, **SL/CP** justifications, **outlist** (non-monotonic), **nogoods**, **cascade/restoration**, **dependency-directed backtracking**, **challenge/defend**. All vocabulary below maps to it. |
| 2 | `CLAUDE.md` | Repo-local contributor contract. Almost certainly documents the `reasons` vs `reasons_lib` split and dev/test conventions. Read before anything else below. |
| 3 | `CHANGELOG.md` | Chronology of feature landings (nogoods, outlist, dialectical, merkle, review/premise agents, MCP). Explains *why* modules like `repair_premises.py`, `report_gated.py`, `review_justifications.py`, `mind_service.py` exist and when they were added. |
| 4 | `tests/conftest.py` | The fastest way to learn the data model. Fixtures will reveal the actual node/justification/nogood shapes, DB setup, and helper factories that every test shares. |
| 5 | `reasons/storage.py` | **SQLite persistence layer + schema.** Almost every module imports this. Nodes table (status, premise flag, metadata), justifications (type, inlist, outlist), dependents edges, nogoods, and the propagation audit log all live here. Schema *is* the domain model. |
| 6 | `reasons/network.py` | The TMS graph abstraction: node creation, justification wiring, dependent registration, outlist handling. If `storage.py` is state, `network.py` is structure. Note: `network.py` also appears with a `test_network_cache.py` and `test_network.py` — read to see if this is a PubSub/agent network or the belief graph (the naming is overloaded). |
| 7 | `reasons/derive.py` | The **propagation engine**: evaluates SL (inlist IN ∧ outlist OUT) and CP (assumption consistency) justifications, walks dependents, performs cascade retraction and restoration. Governs whether a node is IN/OUT. `derive_budget` and `any_mode` tests hang off this. |
| 8 | `reasons/cli.py` (`main`) | Primary entry point (`reasons`). ~30 subcommands ↔ ~30 modules; the CLI dispatch table is a **module inventory in disguise** — use it as a navigation index. |
| 9 | `reasons/api.py` | The programmatic contract other layers use (`llm`, `ask`, `review`, `mind_service`, `mcp_server`). Consolidated import surface — tells you what's considered public vs internal. |
| 10 | `reasons/contradictions.py` (+ `repair.py`, `repair_premises.py`) | **Dependency-directed backtracking.** Given a nogood (A B C cannot all be IN), trace antecedent graph and retract the responsible *premise* with minimal disruption. `test_backtracking.py`, `test_migrate_defeaters.py`, `test_classify_defeaters.py`, `test_defeat_with_scope.py` all target this. |
| 11 | `reasons/import_beliefs.py` + `reasons/export_markdown.py` | The **beliefs.md interchange format** (roundtrip). This is how external knowledge bases enter/leave the system; dictates what properties (source, scope, defeater, challenge metadata) survive serialization. `test_roundtrip.py` is the contract. |
| 12 | `reasons/check_stale.py` + `reasons/metadata.py` | **Source freshness subsystem** — node metadata (source path, content hash, scope, access tags) drives `check-stale` and `hash-sources`. `merkle.py` (only in `reasons/`) likely extends this to tree-level verification; explain the asymmetry with `reasons_lib`. |
| 13 | `reasons/review.py` + `propose_update.py` + `report_*.py` | The **agentic layer** over the TMS: LLM-driven justification review, premise repair proposals, and gated reports. Likely wraps `llm.py`, `ask.py`, `cluster.py` (embeddings for dedup/clustering). Understand this to know the "product" beyond the raw engine. |
| 14 | `reasons/mcp_server.py` (`main`) | Second entry point (`reasons-mcp`). Defines how external agents (Claude, other MCP clients) consume the belief store — tool surface = user-facing API in practice. Pairs with `mcp_client.py` and `test_ask_mcp.py`. |
| 15 | `reasons/pg.py` + `reasons/pubsub.py` | Scale-out / coordination: Postgres backend and pub/sub for multi-process or multi-agent sync. Optional extras in `pyproject.toml` (`pg = [...]`). Skip in first pass unless tests (`test_pg_dispatch.py`, `test_pubsub.py`) suggest it's on the hot path. |

---

## 2. Architecture Sketch

**ftl-reasons** is a SQLite-backed **Truth Maintenance System** (Doyle 1979) with a layered architecture: a small dependency-free core (`storage` → `network` → `derive` → `contradictions`) implements belief nodes, SL/CP justifications, non-monotonic outlists, nogoods, and dependency-directed backtracking; above it sits an **interchange layer** for `beliefs.md` round-tripping and source-freshness tracking; and above that a **multi-surface interface layer** (CLI, programmatic `api`, MCP server/client, `mind_service` for higher-order service, optional Postgres+PubSub for scale). Alongside the engine is an **agentic layer** (`llm`, `ask`, `review*`, `propose_update`, `report_*`, `repair*`, `cluster`) that uses LLMs to propose, scrutinize, and repair justifications and premises — effectively using LLMs to maintain a formal belief graph. The near-duplicate `reasons/` vs `reasons_lib/` trees are the biggest packaging anomaly; `reasons/` appears canonical (matches pyproject scripts and has `merkle.py`), while `reasons_lib/` is out of sync and should be treated as a reconciliation task, not a module to study.

---

## 3. Module Map

- **Core TMS engine** — `storage.py`, `network.py`, `derive.py`, `verify.py`, `contradictions.py`, `repair.py`, `repair_premises.py`, `merkle.py`
- **Interchange & freshness** — `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `check_stale.py`, `metadata.py`, `compact.py`, `build_wiki.py`, `hf.py`
- **Agentic / LLM layer** — `llm.py`, `ask.py`, `cluster.py`, `review.py`, `review_justifications.py`, `review_premises.py`, `propose_update.py`, `report_belief.py`, `report_gated.py`, `derive.py` (also serves this use via `derive_budget`), `repair.py`
- **Interfaces** — `cli.py` (main tool), `api.py` (Python API), `mcp_server.py` + `mcp_client.py` (agent tool protocol), `mind_service.py` (service wrapper), `pg.py` + `pubsub.py` + `network.py`-adjacent (coordination / scale)
- **Tests** — grouped by domain: propagation (`test_derive*`, `test_any_mode`, `test_dialectical`), nogoods/backtracking (`test_backtracking`, `test_nogood_id`, `test_classify_defeaters`, `test_defeat_*`, `test_migrate_defeaters`), integrity (`test_dangling_dependents`, `test_dependents_integrity`, `test_outlist`, `test_remove_justification`, `test_add_justification`), interchange (`test_import_*`, `test_export_*`, `test_roundtrip`, `test_pin_sources`, `test_source_type`), agentic (`test_review*`, `test_repair*`, `test_propose_update`, `test_research`, `test_sync_agent`, `test_summarize`)
- **Meta / tooling (non-code)** — `.forge/` (proposed-entries, report), `.claude/skills/reasons`, `costs/*.json` (LLM cost logs), `.github/workflows/tests.yml`

---

## 4. Exploration Strategy

1. **Semantics first** — `README.md`, `CLAUDE.md`, `CHANGELOG.md`. Learn the TMS vocabulary and the reasons/reasons_lib story before reading a line of code.
2. **Data model via tests** — `tests/conftest.py`, then skim `tests/test_storage.py` to see the schema as fixtures.
3. **Core engine, bottom-up** — `storage.py` → `network.py` → `derive.py`. This is the propagation heart; nothing else makes sense until this is clear.
4. **Persistence-adjacent invariants** — `verify.py`, `contradictions.py` + `repair.py`, then `tests/test_backtracking.py`, `test_nogood_id.py`, `test_dialectical.py` side-by-side.
5. **CLI as index** — `cli.py` (its dispatch dict lists every command ↔ module pair), then `api.py` for the programmatic equivalent.
6. **Interchange & freshness** — `import_beliefs.py` + `export_markdown.py` + `metadata.py` + `check_stale.py`, cross-checked with `test_roundtrip.py`.
7. **Agentic layer** — `llm.py`, `ask.py`, `review*.py`, `propose_update.py`, `report_gated.py`, `repair_premises.py`. Read last; it *consumes* the core.
8. **Interfaces & scale-out** — `mcp_server.py` / `mcp_client.py`, `mind_service.py`, then `pg.py` / `pubsub.py` only if the deployment story matters.
9. **Resolve duplication** — diff `reasons/` vs `reasons_lib/` (start with `merkle.py` and `__init__.py`) and decide canonical home before contributing.

---

## Topics to Explore

- [file] `README.md` — Canonical TMS vocabulary: IN/OUT, SL/CP, outlist, nogood, cascade, backtracking, challenge/defend.
- [file] `CLAUDE.md` — Repo conventions; likely the only place the `reasons` vs `reasons_lib` split is explained.
- [file] `CHANGELOG.md` — Feature chronology; explains why agentic and repair modules exist and when `merkle.py` diverged.
- [file] `tests/conftest.py` — Fastest reveal of the node/justification/nogood data model via shared fixtures.
- [file] `reasons/storage.py` — SQLite schema = domain model; the layer everything imports.
- [file] `reasons/network.py` — Graph structure: node creation, justification wiring, dependents, outlist registration.
- [file] `reasons/derive.py` — Propagation engine; determines IN/OUT from SL/CP/outlist and drives cascade/restoration.
- [file] `reasons/cli.py:main` — Primary entry point; subcommand dispatch is an implicit module index.
- [file] `reasons/api.py` — Public programmatic contract consumed by the agentic and MCP layers.
- [general] `dependency-directed-backtracking` — How nogoods trace antecedents to retract the responsible *premise* with minimal disruption (see `contradictions.py`, `repair.py`, `test_backtracking.py`).
- [file] `reasons/import_beliefs.py` — beliefs.md import format; governs what metadata survives round-trips (see `test_roundtrip.py`).
- [file] `reasons/export_markdown.py` — Inverse of import; canonical serializer for the interchange format.
- [file] `reasons/check_stale.py` — Source-freshness check; entry to `metadata.py` hashing and `merkle.py`.
- [file] `reasons/review.py` — Agentic justification review; entry into the LLM-backed layer (`llm`, `ask`, `cluster`, `propose_update`, `report_*`).
- [file] `reasons/mcp_server.py:main` — Second entry point (`reasons-mcp`); defines the external agent tool surface.
- [general] `reasons-vs-reasons_lib` — Reconcile the near-duplicate packages; decide canonical home before any change (start by diffing `merkle.py`, `__init__.py`, `CHANGELOG.md`).
