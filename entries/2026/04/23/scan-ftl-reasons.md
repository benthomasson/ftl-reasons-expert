# Scan: ftl-reasons

**Date:** 2026-04-23
**Time:** 16:29

Now I have a complete picture of the codebase. Here's my analysis:

---

## Architecture Sketch

ftl-reasons is a Python implementation of Doyle's (1979) Truth Maintenance System (TMS). The architecture is a clean three-layer stack: **data model** (`__init__.py` — `Node`, `Justification`, `Nogood` dataclasses), **engine** (`network.py` — the `Network` class implementing BFS propagation, retraction cascades, restoration, nogoods, and dialectical argumentation), and **persistence** (`storage.py` — SQLite with FTS5). The `api.py` module provides a functional Python API on top of these (open-db, operate, save, close pattern via `_with_network` context manager), and `cli.py` is a thin argparse wrapper that formats `api` return dicts for terminal output. Supporting modules handle import/export (`import_beliefs.py`, `import_agent.py`, `export_markdown.py`), LLM-assisted derivation (`derive.py`), staleness detection (`check_stale.py`), and token-budgeted summaries (`compact.py`).

## Module Map

| Responsibility | Files |
|---|---|
| **Data model** | `__init__.py` |
| **Core engine** | `network.py` |
| **Persistence** | `storage.py` |
| **Public API** | `api.py` |
| **CLI** | `cli.py` |
| **Import/Export** | `import_beliefs.py`, `import_agent.py`, `export_markdown.py` |
| **LLM integration** | `derive.py` |
| **Staleness & hashing** | `check_stale.py` |
| **Context injection** | `compact.py` |
| **Tests** | `tests/` (18 test files, 211+ tests) |
| **CI** | `.github/workflows/tests.yml` |
| **Code expert state** | `.code-expert/`, `entries/`, `reviews/` |

## Critical Files

1. **`reasons_lib/__init__.py`** — Shared data model (`Node`, `Justification`, `Nogood`). Every other module imports from here.
2. **`reasons_lib/network.py`** — The TMS engine: propagation, retraction cascades, restoration, nogoods with dependency-directed backtracking, challenge/defend, supersession, summarize. The intellectual core.
3. **`reasons_lib/storage.py`** — SQLite persistence with schema, full save/load, FTS5 index. All state durability flows through here.
4. **`reasons_lib/api.py`** — Functional API layer (~1500 lines). Namespace support, what-if simulation, deduplication, search, and the `_with_network` pattern that ensures atomic operations.
5. **`reasons_lib/cli.py`** — Entry point (`reasons` command). 35+ subcommands. Demonstrates the full feature surface.
6. **`reasons_lib/import_agent.py`** — Multi-agent belief federation: namespacing, active/inactive relay nodes, outlist-based kill switches, sync with "remote wins" semantics.
7. **`reasons_lib/derive.py`** — LLM-driven derivation: prompt building, proposal parsing, validation against existing network, exhaustive multi-round derivation.
8. **`reasons_lib/import_beliefs.py`** — Markdown parser for beliefs.md format. Topological sort for dependency ordering.
9. **`reasons_lib/export_markdown.py`** — Reverse of import: network → beliefs.md. Round-trip fidelity matters.
10. **`reasons_lib/compact.py`** — Token-budgeted summaries for LLM context injection. Priority ordering (nogoods > OUT > IN by dependents).
11. **`reasons_lib/check_stale.py`** — Source hash comparison to detect when beliefs are outdated.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Core data model: Node, Justification, Nogood dataclasses shared across all modules
- [file] `reasons_lib/network.py` — TMS engine: propagation, retraction cascades, restoration, BFS, truth computation
- [function] `reasons_lib/network.py:_propagate` — BFS cascade through dependents — the central algorithm
- [function] `reasons_lib/network.py:_justification_valid` — Truth computation: SL/CP with inlist AND and outlist negation
- [function] `reasons_lib/network.py:add_nogood` — Dependency-directed backtracking to resolve contradictions
- [function] `reasons_lib/network.py:challenge` — Dialectical argumentation via outlist mechanism
- [file] `reasons_lib/storage.py` — SQLite persistence: schema, save/load, FTS5 index, dependent rebuild
- [file] `reasons_lib/api.py` — Functional API: _with_network pattern, namespace support, what-if simulation, dedup
- [file] `reasons_lib/import_agent.py` — Multi-agent federation: namespacing, active/inactive relay, sync semantics
- [file] `reasons_lib/derive.py` — LLM-driven derivation: prompt building, proposal parsing, exhaustive mode
- [file] `reasons_lib/compact.py` — Token-budgeted summary with priority ordering and summary node compression
- [general] `outlist-semantics` — Non-monotonic reasoning: how outlists enable "believe X unless Y", supersession, and challenges
- [general] `multi-agent-federation` — How agents are namespaced, the active/inactive relay pattern, and kill-switch cascades
- [file] `reasons_lib/cli.py` — CLI entry point: 35+ subcommands, derive orchestration with async model invocation

