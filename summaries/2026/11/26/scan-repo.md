---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Scan: repo

## Critical Files

1. `README.md` — Defines the domain model: nodes, IN/OUT truth values, SL/CP justifications, outlists, nogoods, retraction cascades, restoration, and dependency-directed backtracking. This is the fastest way to understand the intended semantics before reading code.

2. `pyproject.toml` — Shows the package layout, entry points (`reasons.cli:main`, `reasons.mcp_server:main`), optional dependency groups, and that both `reasons` and `reasons_lib` are packaged. Critical for understanding how the app is invoked and which integrations are optional.

3. `reasons/cli.py` — Primary user-facing entry point. Likely dispatches commands such as `add`, `retract`, `assert`, `nogood`, `explain`, `trace`, `import-beliefs`, etc. Best place to map commands to core services.

4. `reasons/api.py` — Likely the programmatic façade over the TMS engine. Important for understanding reusable operations separate from CLI formatting.

5. `reasons/metadata.py` — Likely contains shared node/justification metadata, enums, or dataclasses. Shared types are usually the key to understanding storage, propagation, and verification.

6. `reasons/storage.py` — Core persistence layer. Likely defines the SQLite schema, node/justification CRUD, dependency storage, and transaction boundaries. Essential for understanding how the belief network is represented.

7. `reasons/network.py` — Likely the belief/dependency network and propagation logic: retraction cascades, restoration, dependency traversal, and truth-value recomputation. This is probably the conceptual heart of the TMS.

8. `reasons/derive.py` — Likely handles derived nodes and justification validity. Important for understanding how `--sl`, `--cp`, and `--unless` justifications are constructed and evaluated.

9. `reasons/verify.py` — Likely validates invariants such as node status, justification consistency, dependency integrity, and propagation correctness. High-value for understanding expected TMS correctness rules.

10. `reasons/contradictions.py` — Likely nogood detection and contradiction classification. Important for dependency-directed backtracking and dialectical behavior.

11. `reasons/repair.py` — Likely implements repair/backtracking after contradictions or invalid justifications. Core to the “retract the responsible premise with minimal disruption” behavior.

12. `reasons/mcp_server.py` — Secondary entry point exposing the system through MCP. Important if the project is used by LLM/agent tooling; reveals the external tool surface and API contracts.

13. `reasons/pg.py` — Optional PostgreSQL backend. Important only after understanding `storage.py`, but it reveals whether persistence is abstracted or whether SQLite-specific assumptions leak into the core.

14. `tests/conftest.py` — Test fixtures and shared setup. Likely shows how databases, nodes, justifications, and temporary networks are constructed, making the test suite much easier to read.

15. `reasons_lib/__init__.py` — The repository contains a near-duplicate `reasons_lib` package alongside `reasons`, and `pyproject.toml` packages both. The tree suggests drift, e.g. `reasons/merkle.py` appears absent from `reasons_lib`. This file should be inspected to determine whether it is a compatibility shim, stale fork, or alternate implementation.

## Architecture Sketch

The project is a Truth Maintenance System with two main entry points: a CLI (`reasons.cli:main`) and an MCP server (`reasons.mcp_server:main`). Those interfaces likely call into shared API/domain modules that manipulate a persisted belief network stored primarily in SQLite, with optional PostgreSQL support. The core engine is split across storage, metadata/shared types, dependency-network propagation, derivation/justification handling, verification, contradiction/nogood handling, and repair/backtracking modules. Optional modules provide LLM/agent integration, imports/exports, embeddings/clustering, review workflows, and MCP tooling. The duplicate `reasons_lib` package should be treated as an architectural risk until its relationship to `reasons` is confirmed.

## Module Map

- **Core TMS engine**
  - `reasons/storage.py`, `reasons/metadata.py`, `reasons/network.py`, `reasons/derive.py`, `reasons/verify.py`, `reasons/contradictions.py`, `reasons/repair.py`, `reasons/repair_premises.py`
  - Likely owns nodes, justifications, truth values, propagation, nogoods, backtracking, and invariants.

- **Interfaces / orchestration**
  - `reasons/cli.py`, `reasons/api.py`, `reasons/mcp_server.py`, `reasons/mcp_client.py`, `reasons/ask.py`, `reasons/mind_service.py`, `reasons/pubsub.py`
  - CLI, programmatic API, MCP integration, agent-facing services, and messaging.

- **Persistence**
  - `reasons/storage.py`, `reasons/pg.py`
  - SQLite main store plus optional PostgreSQL backend.

- **Import / export / reporting**
  - `reasons/import_agent.py`, `reasons/import_beliefs.py`, `reasons/export_card.py`, `reasons/export_markdown.py`, `reasons/build_wiki.py`, `reasons/report_belief.py`, `reasons/report_gated.py`
  - Conversion to/from external belief formats, markdown, cards, wiki pages, and reports.

- **Review / maintenance / reasoning extensions**
  - `reasons/review.py`, `reasons/review_justifications.py`, `reasons/review_premises.py`, `reasons/propose_update.py`, `reasons/check_stale.py`, `reasons/compact.py`, `reasons/cluster.py`, `reasons/contradictions.py`
  - Workflow-oriented operations around belief quality, staleness, summarization, clustering, and contradictions.

- **External integrations**
  - `reasons/llm.py`, `reasons/hf.py`, `reasons/cluster.py`, `reasons/merkle.py`, `reasons/network.py`
  - LLM/Hugging Face/embedding/clustering integrations; `network.py` should be verified because its name could mean belief network rather than external networking.

- **Duplicate / compatibility package**
  - `reasons_lib/`
  - Mirrors most of `reasons/`, but the file listing differs. Determine whether this is legacy, generated, or compatibility code.

- **Tests**
  - `tests/`
  - Rich behavioral specification. Especially important: `test_storage.py`, `test_verify.py`, `test_derive.py`, `test_contradictions.py`, `test_repair.py`, `test_backtracking.py`, `test_dialectical.py`, `test_outlist.py`, `test_review.py`, `test_mcp_server.py`.

- **Project infrastructure**
  - `.github/workflows/tests.yml`, `pyproject.toml`, `CLAUDE.md`, `CHANGELOG.md`, `.forge/`, `costs/`
  - CI, packaging, agent instructions, release history, and auxiliary generated reports.

## Exploration Strategy

1. Read `README.md` and `pyproject.toml` to establish the domain model and execution entry points.
2. Trace `reasons/cli.py:main` and `reasons/mcp_server.py:main` to see which core modules they call.
3. Read `reasons/metadata.py` and `reasons/storage.py` to understand the persisted data model and shared types.
4. Follow into `reasons/network.py`, `reasons/derive.py`, and `reasons/verify.py` to understand propagation and justification validity.
5. Read `reasons/contradictions.py` and `reasons/repair.py` to understand nogoods and dependency-directed backtracking.
6. Read `tests/conftest.py`, then the core tests such as `test_storage.py`, `test_derive.py`, `test_verify.py`, `test_backtracking.py`, `test_outlist.py`, and `test_dialectical.py`.
7. Only then explore optional integrations such as `reasons/pg.py`, `reasons/llm.py`, `reasons/mcp_server.py`, imports/exports, and review workflows.
8. Finally, inspect `reasons_lib/__init__.py` and compare `reasons_lib` against `reasons` to resolve the duplication risk.

## Topics to Explore

- [file] `README.md` — Defines the TMS semantics, core vocabulary, and expected CLI behavior.
- [file] `pyproject.toml` — Shows package boundaries, entry points, optional dependencies, and the duplicated package layout.
- [function] `reasons/cli.py:main` — Primary CLI dispatcher and best starting point for command-to-module mapping.
- [function] `reasons/mcp_server.py:main` — Secondary entry point exposing the system to MCP/agent tooling.
- [file] `reasons/api.py` — Programmatic façade over the TMS operations.
- [file] `reasons/metadata.py` — Likely shared node/justification types and status enums.
- [file] `reasons/storage.py` — Persistence schema and CRUD operations for nodes, justifications, and dependencies.
- [file] `reasons/network.py` — Likely dependency graph propagation, retraction cascades, and restoration.
- [file] `reasons/derive.py` — Derivation and justification construction/evaluation.
- [file] `reasons/verify.py` — Invariant checks and truth-maintenance correctness rules.
- [file] `reasons/contradictions.py` — Nogood detection and contradiction classification.
- [file] `reasons/repair.py` — Dependency-directed backtracking and repair logic.
- [file] `tests/conftest.py` — Shared fixtures and database/network setup for the behavioral test suite.
- [general] `reasons-vs-reasons_lib-duplication` — Determine whether `reasons_lib` is a compatibility shim, stale copy, or active alternate implementation.
- [file] `reasons/pg.py` — Optional PostgreSQL backend and evidence of storage abstraction boundaries.
