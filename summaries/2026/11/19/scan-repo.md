---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Scan: repo

Initial read: `reasons/` is the active package. `reasons_lib/` is a near-duplicate mirror and is missing at least `merkle.py`; treat it as legacy/compat until proven otherwise.

## Critical Files

- `README.md` — Best domain-level description of the TMS model, CLI behavior, retraction/restoration, nogoods, outlists, and challenge/defend workflows.
- `pyproject.toml` — Confirms package layout, entry points (`reasons.cli:main`, `reasons.mcp_server:main`), optional dependency boundaries, and that both `reasons` and `reasons_lib` are packaged.
- `reasons/__init__.py` — Quick way to see the public package surface and any exported API/version conventions.
- `reasons/cli.py` — Primary entry point. Maps user commands onto the core API/storage and reveals the intended workflow.
- `reasons/storage.py` — Likely the most important core file: persistence schema and operations for nodes, justifications, nogoods, truth values, and audit history.
- `reasons/api.py` — Likely the shared programmatic API used by CLI/MCP/other modules; central orchestration layer.
- `reasons/derive.py` — Likely core derivation/propagation logic; tests for `derive`, `derive_budget`, and backtracking suggest this is close to the TMS engine.
- `reasons/contradictions.py` — Central to nogood handling, contradiction detection, and dependency-directed backtracking.
- `reasons/verify.py` — Likely invariant/consistency checking for the belief graph; useful for understanding what “correct” state means.
- `reasons/network.py` — Likely dependency-network/graph abstraction or cache layer; confirm whether it is core graph logic or a supporting network/cache helper.
- `reasons/mcp_server.py` — Second entry point. Shows how the TMS is exposed as an MCP service/tool surface.
- `reasons/llm.py` — Optional LLM integration boundary used by higher-level agent workflows such as ask/review/repair/derive.
- `tests/conftest.py` — Fixtures likely reveal the real data model, test database setup, and common graph scenarios.
- `tests/test_backtracking.py` — Behavioral spec for dependency-directed backtracking and retraction cascades, among the highest-risk core behaviors.

## Architecture Sketch

This is a Python CLI plus MCP service layered over a Doyle-style Truth Maintenance System. The core likely lives in `storage.py` plus engine/orchestration modules such as `api.py`, `derive.py`, `contradictions.py`, and `verify.py`, with SQLite as the default persistence layer and optional Postgres support in `pg.py`. `cli.py` and `mcp_server.py` are adapters over the shared API, while `ask`, `llm`, `review`, `repair`, `import`, `export`, `compact`, and `network` modules provide optional agent/import/export/summary workflows. `reasons_lib/` appears to be a duplicated or legacy copy of `reasons/`; verify against imports and CI before assuming it is live code.

## Module Map

- `reasons/` — Active core package: CLI, storage, TMS logic, MCP server, LLM/agent workflows, import/export, verification.
- `reasons_lib/` — Near-duplicate package mirror; likely legacy/compat or generated copy. Missing at least `merkle.py`.
- `tests/` — Executable behavioral specifications organized by feature/command; core tests reveal intended TMS semantics.
- `.github/workflows/tests.yml` — CI entry point; useful for knowing supported dependency modes and test invocation.
- `.claude/skills/reasons` — Tool-specific agent instructions; lower priority for core architecture.
- Root docs/config — `README.md`, `CLAUDE.md`, `CHANGELOG.md`, `pyproject.toml`.

## Exploration Strategy

1. Start with `README.md` and `pyproject.toml` to understand the domain, commands, packaging, and entry points.
2. Read `tests/conftest.py` plus `tests/test_storage.py`, `tests/test_api.py`, and `tests/test_backtracking.py` to learn the expected data model and invariants.
3. Read `reasons/storage.py`, then `reasons/api.py`, `reasons/derive.py`, `reasons/contradictions.py`, and `reasons/verify.py` as the core TMS path.
4. Read `reasons/cli.py` and `reasons/mcp_server.py` to see how the core is exposed.
5. Explore optional workflows only after the core is clear: `llm.py`, `ask.py`, `network.py`, `review*.py`, `repair*.py`, import/export/compact modules.
6. Diff `reasons/` against `reasons_lib/` to confirm whether the latter is legacy, compatibility-only, or still used.

## Topics to Explore

- [file] `README.md` — Domain model, CLI workflows, and intended TMS semantics.
- [file] `pyproject.toml` — Packaging, entry points, optional dependencies, and the active package layout.
- [file] `reasons/__init__.py` — Public package surface and exported conveniences.
- [function] `reasons/cli.py:main` — Primary CLI entry point and command dispatch.
- [file] `reasons/storage.py` — Persistence schema and core node/justification/nogood state.
- [file] `reasons/api.py` — Shared programmatic API and orchestration layer.
- [file] `reasons/derive.py` — Derivation/propagation logic and likely retraction/restoration mechanics.
- [file] `reasons/contradictions.py` — Nogood handling and dependency-directed backtracking.
- [file] `reasons/verify.py` — Integrity checks and TMS invariants.
- [file] `reasons/network.py` — Dependency-network/graph abstraction or supporting cache; confirm its role.
- [function] `reasons/mcp_server.py:main` — MCP service entry point and tool exposure.
- [file] `reasons/llm.py` — Optional LLM integration boundary for agent workflows.
- [file] `tests/conftest.py` — Test fixtures that reveal the real data model and setup.
- [file] `tests/test_backtracking.py` — Behavioral spec for backtracking and cascade correctness.
- [general] `package-duplication` — Determine why `reasons_lib/` mirrors `reasons/` and whether it is legacy or still consumed.
