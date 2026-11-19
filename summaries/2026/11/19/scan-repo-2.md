---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Scan: repo

# Initial Reconnaissance

Before diving in, a few structural observations that shape the whole exploration:

- **Two near-identical packages** (`reasons/` and `reasons_lib/`) are both declared in `pyproject.toml`, but only `reasons` is wired to entry points. `merkle.py` appears in `reasons/` but **not** in `reasons_lib/` — this asymmetry (a recently-added module) strongly suggests `reasons_lib` is a vendored/legacy duplicate that's being retired. Understanding this early prevents reading every file twice.
- **Zero required dependencies.** The core TMS engine is stdlib-only (implies SQLite via `sqlite3` in `storage.py`). Everything else (`pg`, `mcp`, `llm`, `cluster`, `openai`, `vertex`) is an optional extra — so the architecture is a small pure core with pluggable interface/backends around it.
- **Test file names reveal features with no dedicated module** (e.g., `test_backtracking`, `test_dialectical`, `test_research`, `test_summarize`, `test_pin_sources`, `test_hide_out`, `test_access_tags`, `test_nogood_id`). These features are threaded through existing modules rather than isolated, so tests are the fastest way to discover where behavior lives.
- **`check_stale_issue25`, `migrate_defeaters`, `classify_defeaters`** — past-schema evolution is baked into the repo history; `verify.py` / `repair*.py` / `review_justifications.py` likely hold migration + repair logic.

---

## Critical Files

| File | Why |
|---|---|
| `reasons/cli.py` | Primary entry point (`reasons = reasons.cli:main`). Wires every command — the highest-leverage summary of the feature set. |
| `reasons/api.py` | Programmatic API surface; likely the seam both `cli.py` and `mcp_server.py` build on. |
| `reasons/storage.py` | SQLite persistence + schema; shared node/justification/nogood types. Everything reads/writes here. |
| `reasons/derive.py` | The actual TMS propagation/retraction/restoration algorithm. Core domain logic. |
| `reasons/contradictions.py` | Nogoods and dependency-directed backtracking (the distinguishing algorithm from the README). |
| `reasons/verify.py` | Integrity/consistency checks; likely where dependents-integrity + dangling-dependents fixes live. |
| `reasons/mcp_server.py` | Second entry point (`reasons-mcp`); how agents talk to the TMS over MCP. |
| `reasons/import_beliefs.py` | Ingestion pipeline for the `beliefs.md` format — a first-class use case. |
| `reasons/compact.py` | Token-budgeted summarization; represents the "context injection" output path. |
| `reasons/merkle.py` | Content-addressing — the one module unique to `reasons/`, likely used for sync/dedup across the distributed layer. |
| `reasons/network.py` + `pg.py` + `pubsub.py` | Distributed/Postgres backend trio; explains the pg/dispatch tests. |
| `CLAUDE.md` | Project-specific conventions for AI agents — often richer than README about design intent. |

---

## Architecture Sketch

A stdlib-only **core engine**: `storage.py` provides SQLite-backed persistence of nodes/justifications/nogoods, and `derive.py` (with `contradictions.py` and `verify.py`) implements Doyle's truth-maintenance propagation, nogood backtracking, and invariant checks. Around that core sit **interface layers** — `cli.py` (human), `mcp_server.py` (agent), `api.py`/`mind_service.py` (programmatic) — that translate commands into storage+derive operations. **Ingest/egress** (`import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `compact.py`) bridges the graph to human/LLM-consumable formats. **Optional distributed/backends** (`pg.py`, `network.py`, `pubsub.py`, `merkle.py`, `hf.py`) extend the local SQLite story to Postgres, remote sync, and HF hosting, and **LLM-assisted utilities** (`llm.py`, `review*.py`, `propose_update.py`, `cluster.py`, `api.py` in the langchain sense) layer automated belief hygiene on top. `reasons_lib/` appears to be a legacy shadow copy of the same code, minus the newer `merkle.py`.

---

## Module Map

- **Core engine** — `storage.py`, `derive.py`, `contradictions.py`, `verify.py`, `metadata.py`, `merkle.py`
- **Interfaces** — `cli.py` (main), `mcp_server.py`, `mcp_client.py`, `api.py`, `mind_service.py`, `ask.py`, `report_belief.py`, `report_gated.py`
- **Ingest / Egress** — `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `compact.py`, `build_wiki.py`
- **Belief hygiene / review** — `review.py`, `review_justifications.py`, `review_premises.py`, `repair.py`, `repair_premises.py`, `propose_update.py`, `check_stale.py`, `cluster.py`, `derive.py` sub-flows
- **Distributed / storage backends** — `pg.py`, `network.py`, `pubsub.py`, `hf.py`
- **LLM adapters** — `llm.py`, `import_agent.py`, `api.py` (when using `[api]`/`[llm]` extras)
- **Dialectic layer** — `contradictions.py`, `review.py`, `propose_update.py` (challenge/defend/nogood flows)
- **Config & docs** — `pyproject.toml`, `README.md`, `CLAUDE.md`, `CHANGELOG.md`, `.github/workflows/tests.yml`
- **Tests** — flat `tests/` directory, one file per feature; `conftest.py` is the fixture hub

---

## Topics to Explore

- [file] `pyproject.toml` — Entry points, extras, and packaging tell you which modules are "core" vs pluggable; also confirms the `reasons` + `reasons_lib` dual-package situation.
- [file] `CLAUDE.md` — Project conventions and design intent; often more explicit than README about invariants the code must uphold.
- [function] `reasons/cli.py:main` — The command dispatcher; enumerates every feature and where each delegates. Start here to map the surface area.
- [file] `reasons/api.py` — The stable programmatic API. Seeing what it (re-)exports reveals which modules are considered public, and how `cli.py`/`mcp_server.py` share logic.
- [file] `reasons/storage.py` — The SQLite schema and the concrete node/justification/nogood dataclasses; everything downstream depends on these shapes.
- [file] `reasons/derive.py` — The core propagation algorithm: retraction cascade, restoration, multiple justifications, outlist (non-monotonic) handling. This is the reason the project exists.
- [file] `reasons/contradictions.py` — Nogoods + dependency-directed backtracking; compare with the `test_backtracking` / `test_nogood_id` tests to confirm the intended semantics.
- [file] `reasons/verify.py` — Integrity invariants (dependents integrity, dangling dependents, hide-out) that the rest of the code assumes hold.
- [file] `reasons/mcp_server.py` — The agent-facing entry point; shows how the TMS is exposed over MCP and which subset of commands agents get.
- [file] `reasons/import_beliefs.py` — The beliefs.md → graph ingest pipeline, including auto-detection of `nogoods.md`; a canonical "real-world" entry into the system.
- [file] `reasons/compact.py` — Token-budgeted summary generation — the controlled-output path that connects the graph to LLM contexts.
- [file] `reasons/merkle.py` — Content-hash layer unique to `reasons/`; likely underpins dedup/sync in `network.py` and the reason the two packages diverge.
- [general] `reasons-vs-reasons_lib-duplication` — Reconcile the two packages: diff `reasons/` against `reasons_lib/` to confirm whether one is vendored legacy, and which is authoritative.
- [file] `tests/conftest.py` — Test fixtures reveal how a "network" is constructed in memory, providing the fastest mental model of the data model before reading storage.py in depth.
- [general] `non-monotonic-justification-semantics` — The inlist/outlist justification model (default reasoning via `--unless`) is threaded through `derive.py`, `review.py`, and the dialectic challenge/defend flow; trace it end-to-end.

**Suggested exploration order:** `pyproject.toml` → `CLAUDE.md` → `reasons/cli.py:main` → `reasons/api.py` → `tests/conftest.py` → `reasons/storage.py` → `reasons/derive.py` → `reasons/contradictions.py` → `reasons/verify.py` → `reasons/import_beliefs.py` → `reasons/mcp_server.py` → `reasons/compact.py` → `reasons/merkle.py` → diff against `reasons_lib/`. This starts at the entry surface, drops to the shared schema, reaches the algorithmic core, then fans out to the interface and distributed layers.
