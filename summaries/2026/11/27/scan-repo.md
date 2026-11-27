---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Scan: repo

# Initial Reconnaissance: `ftl-reasons`

## Summary

This repo implements a **Truth Maintenance System (Doyle 1979)** — a belief network with automatic retraction cascades, restoration, and dependency-directed backtracking. It ships a **CLI** (`reasons`) and an **MCP server** (`reasons-mcp`). The biggest structural surprise is that the code exists **twice**: `reasons/` and `reasons_lib/` are near-identical mirrors (topics 14–15 flag this), and `reasons_lib/` is *missing* `merkle.py` — a strong signal of drift worth understanding early.

The heap has a small, dependency-free core (`storage`, `api`, `derive`, `verify`) and a large set of **optional integration layers** (LLM, Postgres, clustering, HuggingFace, network, pubsub, review flows) that are only pulled in via extras.

---

## 1. Critical Files

| File | Why it matters |
|------|----------------|
| `reasons/cli.py` | **Primary entry point** (`reasons = reasons.cli:main`). It enumerates every user-facing operation and shows how the engine is orchestrated. |
| `reasons/api.py` | Likely the **shared programmatic API** consumed by CLI, MCP server, and tests — the real "public contract." |
| `reasons/storage.py` | **The heart of the TMS.** Almost certainly defines `Node`, `Justification`, `Nogood`, the SQLite schema, and the propagate/retract/restore machinery. |
| `reasons/derive.py` | Derivation logic — turns justifications into IN/OUT truth values; drives SL/CP semantics. |
| `reasons/verify.py` | Consistency/nogood checking — decides whether the current IN-set is coherent. |
| `reasons/contradictions.py` | Contradiction detection + **dependency-directed backtracking** (the marquee algorithm from the README). |
| `reasons/mcp_server.py` | **Second entry point** (`reasons-mcp`); exposes the engine to agents via Model Context Protocol. |
| `tests/conftest.py` | Canonical fixtures — the fastest way to learn how to instantiate the engine and what invariants tests assume. |
| `tests/test_storage.py` | Behavioral spec for propagation, cascades, restoration, multiple justifications, diamond deps — the de-facto model of the core semantics. |
| `reasons/compact.py` | Token-budgeted summary output — the "context injection into an LLM" use case, likely a primary motivation for the whole project. |
| `reasons/check_stale.py` | Source-file hash freshness — connects beliefs back to real files; explains the `hash-sources` and `check-stale` commands. |
| `reasons/import_beliefs.py` / `export_markdown.py` | Interop with the external `beliefs.md` registry format (README shows round-tripping). |
| `CLAUDE.md` | Repo-specific conventions/instructions — read after README for how the maintainers intend the code to be navigated. |
| `reasons_lib/api.py` (sample) | Representative of the **mirrored duplicate package**; compare against `reasons/api.py` to quantify divergence. |

---

## 2. Architecture Sketch

`ftl-reasons` is a **layered, dependency-free core** wrapped by optional integration surfaces. The core is a persistent belief graph stored in SQLite (`storage.py`), where nodes carry SL/CP justifications with in/out-lists; a propagation engine computes truth values and cascades retraction/restoration, while `contradictions.py` + `verify.py` implement nogood detection and dependency-directed backtracking. `api.py` presents a stable programmatic surface that both `cli.py` and `mcp_server.py` bind to. Everything else (`llm`, `pg`, `cluster`, `hf`, `network`, `review*`, `repair*`, `mind_service`, `pubsub`) is an **optional adapter** gated behind extras — so the true "cognitive core" is small (~8 modules) and the rest is ecosystem plumbing. The `reasons/` and `reasons_lib/` packages appear to be **the same source shipped twice**; that duplication is the main architectural risk to resolve.

---

## 3. Module Map

- **Core engine (must-read):** `storage.py`, `api.py`, `derive.py`, `verify.py`, `contradictions.py`
- **Entry points / transport:** `cli.py`, `mcp_server.py`, `mcp_client.py`, `pubsub.py`
- **Reasoning overlays:** `ask.py`, `propose_update.py`, `report_belief.py`, `report_gated.py`, `review*.py`, `repair*.py`, `check_stale.py`
- **Interop / serialization:** `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `build_wiki.py`, `compact.py`, `merkle.py`
- **Storage/backends:** `storage.py` (SQLite), `pg.py` (Postgres), `network.py`, `mind_service.py`
- **Optional heavy integrations:** `llm.py`, `hf.py`, `cluster.py`, `api.py` (langchain extra), `metadata.py`
- **Tests:** `conftest.py` + ~60 test modules mirroring the module list.
- **Repo meta/tooling (ignore for code understanding):** `.claude/`, `.forge/`, `.github/`, `costs/`.
- **Duplicate mirror:** `reasons_lib/` — same modules as `reasons/` **except missing `merkle.py`**.

---

## 4. Exploration Strategy

1. **Concepts first:** Read `README.md` → `CLAUDE.md` to lock in IN/OUT/SL/CP/outlist/nogood vocabulary.
2. **Public surface:** `reasons/cli.py:main` (commands) and `reasons/api.py` (contract).
3. **Core data model & algorithm:** `reasons/storage.py` — schema, `Node`/`Justification`, and the propagation routine. Then `derive.py`, `verify.py`, `contradictions.py`.
4. **Behavioral ground truth:** `tests/conftest.py` then `tests/test_storage.py`, `test_backtracking.py`, `test_dialectical.py`, `test_any_mode.py`.
5. **Second entry point:** `mcp_server.py` + `mcp_client.py` (how agents call the engine).
6. **Output/context layer:** `compact.py`, `check_stale.py`, `export_markdown.py`, `import_beliefs.py`.
7. **Duplicate package audit:** diff a few `reasons_lib/*.py` against `reasons/*.py` (start with `api.py`, `storage.py`, `cli.py`) to decide which is canonical.
8. **Optional adapters (last):** `llm.py`, `pg.py`, `cluster.py`, `hf.py`, `network.py` — only as needed for a specific feature.

---

## Topics to Explore

- [general] `truth-maintenance-system` — Core vocabulary (IN/OUT, SL/CP, outlist, nogood, retraction cascade) — every other file assumes it.
- [file] `CLAUDE.md` — Repo-specific conventions/guardrails; how the maintainers intend navigation and edits.
- [file] `reasons/cli.py` — Primary entry point; full command surface that maps 1:1 to user-visible features.
- [function] `reasons/cli.py:main` — Command dispatcher; reveals how api/storage/derive are wired together in practice.
- [file] `reasons/api.py` — The true programmatic contract shared by CLI, MCP server, and tests.
- [file] `reasons/storage.py` — Core data model + SQLite persistence + propagation/retract/restore machinery.
- [function] `reasons/storage.py:propagate` — The truth-value recomputation engine that powers cascades and restoration (README's headline behavior).
- [file] `reasons/derive.py` — Derivation of nodes from SL/CP justifications; defines the reasoning semantics.
- [file] `reasons/contradictions.py` — Nogood handling and dependency-directed backtracking (the intellectual core).
- [file] `reasons/verify.py` — Consistency checking of the current IN-set against nogoods.
- [file] `tests/conftest.py` — Canonical fixtures showing how the engine is instantiated and exercised.
- [file] `tests/test_storage.py` — Authoritative behavioral spec for cascades, diamonds, multiple justifications, restoration.
- [file] `reasons/mcp_server.py` — Secondary entry point (`reasons-mcp`); the agent-facing transport.
- [file] `reasons/compact.py` — Token-budgeted context injection output; likely the project's primary end-use case.
- [file] `reasons/check_stale.py` — Source-hash freshness; connects beliefs to real files (`hash-sources`/`check-stale`).
- [general] `reasons-vs-reasons_lib-duplication` — Investigate why two near-identical packages exist and what `merkle.py`'s absence in `reasons_lib` implies (drift risk).
