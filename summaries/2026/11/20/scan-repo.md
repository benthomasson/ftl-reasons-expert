---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Scan: repo

# Initial Reconnaissance: `ftl-reasons`

This is a **Truth Maintenance System** (Doyle-style) implemented twice over — once in `reasons/` and once in `reasons_lib/` — with nearly identical file names. That duplication is the single most important thing to resolve early: it tells you which package is canonical, whether one is a shim/re-export of the other, and where actual logic lives. The `pyproject.toml` wires `reasons.cli:main` as the console entry, but the split is intentionally non-obvious.

## Critical Files

| # | File | Why it matters |
|---|------|----------------|
| 1 | `pyproject.toml` | Declares both `reasons` and `reasons_lib` as packages, the two entry points (`reasons`, `reasons-mcp`), optional-dependency groups (pg, mcp, llm, cluster), and test config. Fastest way to see the intended runtime surface. |
| 2 | `README.md` | Defines the domain vocabulary you'll need to read any code: node, IN/OUT, SL/CP justifications, inlist/outlist, nogood, retraction cascade, dependency-directed backtracking. |
| 3 | `CLAUDE.md` | Usually contains architecture notes, conventions, and agent instructions for this repo — likely a shortcut to the true design story. |
| 4 | `reasons/storage.py` | Almost certainly the schema + persistence layer (nodes, justifications, nogoods, dependents, audit log). Everything else reads/writes through it. |
| 5 | `reasons/network.py` | The dependency graph and propagation engine — the "network" of Doyle's TMS lives here. Central to cascade/restoration semantics. |
| 6 | `reasons/derive.py` | Likely the core justification-evaluation / derivation logic (SL vs CP, inlist/outlist checks). |
| 7 | `reasons/verify.py` | Truth-value computation and validation (`verify`, `propagate`). Where IN/OUT is actually decided per node. |
| 8 | `reasons/contradictions.py` | Nogoods + dependency-directed backtracking — the most algorithmically interesting part, and the feature the README leads with. |
| 9 | `reasons/cli.py` | Main entry point and the largest command surface (`init/add/retract/assert/nogood/explain/trace/challenge/defend/...`). Great map of user-facing operations. |
| 10 | `reasons/api.py` | Programmatic interface used by tests and external consumers — decouples logic from CLI. |
| 11 | `reasons/mcp_server.py` | Second entry point — exposes reasons over the Model Context Protocol. Answers "how is this consumed by agents?" |
| 12 | `reasons/pg.py` | Parallel Postgres backend to `storage.py`. Understanding the storage abstraction boundary matters before touching either. |
| 13 | `tests/conftest.py` | Test fixtures reveal the canonical construction path (which package, which init function) — a cheap way to learn intended usage. |

Honourable mentions once you're past the core: `reasons/llm.py` (Anthropic/Vertex/langfuse), `reasons/compact.py` (token-budgeted context injection — a real product feature), `reasons/import_beliefs.py` / `export_markdown.py` (beliefs.md interchange format), `challenge`/`defend` logic (likely split across `network.py` and `conflicts` handling).

## Architecture Sketch

This is a layered TMS: a **persistence layer** (`storage.py` for SQLite, `pg.py` for Postgres) holds nodes, justifications (SL/CP with in/out lists), nogoods, and an audit log; a **graph/propagation layer** (`network.py`, `derive.py`, `verify.py`) evaluates justifications and cascades truth-value changes across dependents; a **conflict layer** (`contradictions.py`, `repair*.py`) records nogoods and performs dependency-directed backtracking to retract the responsible premise; and a **thin interface layer** (`cli.py`, `api.py`, `mcp_server.py`) exposes everything as commands, SDK calls, or MCP tools. Optional integration modules (`llm.py`, `propose_update.py`, `import_beliefs.py`, `cluster.py`) bolt on LLM-assisted belief maintenance without contaminating the core. The parallel `reasons/` and `reasons_lib/` trees suggest either a packaging workaround (namespace collision / install path) or an in-progress rename — confirming this is a prerequisite to trusting any import you read.

## Module Map

- **Core TMS engine**: `network.py`, `derive.py`, `verify.py`, `contradictions.py`, `repair.py`, `repair_premises.py`
- **Persistence**: `storage.py` (SQLite), `pg.py` (Postgres), `metadata.py`
- **Graph utilities**: `merkle.py` (content hashing), `cluster.py` (semantic clustering)
- **Interface surfaces**: `cli.py` (primary), `api.py` (SDK), `mcp_server.py` + `mcp_client.py` (MCP), `pubsub.py`
- **Interchange / import-export**: `import_beliefs.py`, `import_agent.py`, `export_markdown.py`, `export_card.py`, `build_wiki.py`, `hf.py`
- **Belief hygiene / maintenance**: `check_stale.py`, `compact.py`, `contradictions.py`, `contradictions`-adjacent `propose_update.py`, `report_belief.py`, `report_gated.py`, `ask.py`, `review*.py`
- **LLM integration**: `llm.py`, `propose_update.py`, `derive.py` (partial)
- **Dialectical reasoning**: `challenge` / `defend` — likely split across CLI + `network.py` + `conflicts`-logic
- **Tests**: `tests/` — one file per feature/module, so names double as a functional index
- **Meta**: `.forge/` (proposals/report), `costs/`, `CHANGELOG.md`, `CLAUDE.md`, `.github/workflows/tests.yml`

## Exploration Strategy

1. **Orient first** — `README.md`, then `CLAUDE.md`, then `pyproject.toml`. Lock the vocabulary and entry points before opening source.
2. **Resolve the duplication** — diff `reasons/cli.py` vs `reasons_lib/cli.py` (and a couple of core modules). Decide which tree is canonical; everything downstream depends on this.
3. **Data model** — read `reasons/storage.py` (and skim `reasons/pg.py`) to learn the schema: how are nodes, justifications, inlist/outlist, and nogoods represented? This is the shared "types" layer.
4. **Propagation** — `network.py` → `derive.py` → `verify.py` in that order. Follow one retraction cascade mentally.
5. **Backtracking** — `contradictions.py` (and any `repair*` helpers). This is where the README's "dependency-directed backtracking" claim is earned.
6. **Widen to interfaces** — `cli.py` and `api.py` (map commands → core calls), then `mcp_server.py` (how agents consume it).
7. **Only then** optional modules: `llm.py`, `import_beliefs.py`, `compact.py`, `cluster.py`.
8. **Cross-check via tests** — `tests/conftest.py`, then `test_network.py`/`test_derive.py`/`test_contradictions.py`; the test names are a functional spec.

## Topics to Explore

- [file] `pyproject.toml` — Entry points, dual-package declaration, optional extras, and test config.
- [file] `CLAUDE.md` — Likely the repo's own architecture/conventions primer; read before any source.
- [general] `truth-maintenance-system` — Doyle 1979 vocabulary (IN/OUT, SL/CP, inlist/outlist, nogood, cascade, DDB) that every file assumes.
- [general] `reasons-vs-reasons-lib` — Resolve the duplicated package tree before trusting imports; determines where real logic lives.
- [file] `reasons/storage.py` — Persistence schema for nodes, justifications, nogoods, and audit log; the shared type layer.
- [file] `reasons/network.py` — Dependency graph + propagation engine; retraction cascade and restoration semantics.
- [file] `reasons/derive.py` — Justification evaluation (SL vs CP, inlist/outlist); core derivation rules.
- [file] `reasons/verify.py` — Truth-value computation and validation (`propagate`); where IN/OUT is finalized.
- [file] `reasons/contradictions.py` — Nogood recording + dependency-directed backtracking; the algorithmic headline feature.
- [function] `reasons/cli.py:main` — Primary entry point and dispatch; map every user-facing command to its core call.
- [file] `reasons/api.py` — Programmatic SDK boundary; used by tests and external consumers, decouples logic from CLI.
- [file] `reasons/mcp_server.py` — Second entry point; defines how agents invoke reasons via MCP tools.
- [file] `reasons/pg.py` — Postgres backend parallel to `storage.py`; clarifies the storage abstraction.
- [file] `tests/conftest.py` — Fixtures reveal canonical construction path (which package, which init), cheaply pinning intended usage.
