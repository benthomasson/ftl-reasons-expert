---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Scan: repo

# Recon Report: `ftl-reasons`

## 1. Critical Files

**First, an architectural red flag worth calling out early:** the repo ships *two* near-identical packages — `reasons/` and `reasons_lib/`. They are distinguished only by `merkle.py`, which exists in `reasons/` (and has `test_merkle.py`) but is absent from `reasons_lib/`. `pyproject.toml` lists both under `packages`, but the entry points (`reasons.cli:main`, `reasons.mcp_server:main`) always resolve to `reasons/`. This strongly suggests `reasons_lib/` is a stale fork/legacy copy that should be investigated for divergence (or removed) — but is otherwise not the canonical code.

With that caveat, the canonical package (`reasons/`) breaks down into five layers: entry points, public API, storage/persistence (which almost certainly also houses the propagation/backtracking engine), a domain model, and peripheral integrations (MCP, HF, LangChain, Postgres, network fetch).

**Tier A — start here:**
- `CLAUDE.md` — project-specific instructions/conventions; fastest way to know how the authors intend the code to be used and extended.
- `pyproject.toml` — already visible; note the optional-dependency groups (`mcp`, `api`, `pg`, `cluster`, `llm`, `openai`, `vertex`) which partition the module surface into functional clusters.
- `reasons/__init__.py` — the package façade; what's re-exported here defines the "considered public" surface and tells you which modules are the real core vs. peripheral.

**Tier B — entry points:**
- `reasons/cli.py` — the primary entry point. It parses ~30 subcommands (`add`, `retract`, `assert`, `nogood`, `trace`, `challenge`, `defend`, `derive`, `compact`, `check-stale`, …). Reading `main` and the dispatch table is the single highest-leverage step.
- `reasons/mcp_server.py` — the second entry point (`reasons-mcp`), exposing the belief system to AI agents. Its tool definitions reveal which API operations are considered first-class.

**Tier C — core logic:**
- `reasons/api.py` — public API consumed by both CLI and MCP server; the orchestration layer between commands and storage.
- `reasons/storage.py` — persistence (SQLite) *and*, given the absence of any dedicated `tms.py`/`propagate.py`, likely the propagation engine itself: retraction cascade, restoration, and the audit log (`reasons log`). `test_storage.py` and `test_roundtrip.py` confirm this is where truth-value state lives.
- `reasons/metadata.py` — the node/justification data model and source-metadata schema (source type, access tags, hashes). `test_metadata.py`, `test_metadata_extended.py`, `test_source_type.py`, `test_access_tags.py` all target it, so it's shared types, not just metadata in the "extras" sense.
- `reasons/verify.py` — invariants/consistency checker. These reveal the intended model precisely: what relationships *must* hold (dependents integrity, no dangling dependents, etc.).

**Tier D — TMS algorithms & peripheral:**
- `reasons/contradictions.py` — nogood detection; likely the home of dependency-directed backtracking (`test_backtracking.py`, `test_classify_defeaters.py`, `test_defeat_justification.py`).
- `reasons/derive.py` — automatic derivation, budgeted (`test_derive_budget.py`).
- `reasons/repair.py` / `reasons/repair_premises.py` — repair logic (probably backtracking-adjacent).
- `reasons/merkle.py` — content-addressing/integrity; the file that *distinguishes* the two packages.
- `tests/conftest.py` — fixtures reveal the real DB bootstrap and data model better than the README does.

## 2. Architecture Sketch

This is a layered CLI-backed TMS: `cli.py` (and `mcp_server.py`) parse commands and delegate to `api.py`, which orchestrates against `storage.py` — a SQLite/Postgres-backed store (`pg.py` is the Postgres backend) that holds nodes, justifications, and nogoods, and almost certainly also runs the propagation algorithm (SL/CP validity, retraction cascade, restoration, dependency-directed backtracking). `metadata.py` defines the shared node/justification model and source-metadata schema, so it's the hub both `api.py` and `storage.py` depend on. Peripherals handle import/export (`import_*.py`, `export_*.py`), external fetching (`network.py`, `hf.py` — HTTP with a cache), LLM-assisted operations (`llm.py`, `ask.py`), and dialectical argumentation (`contradictions.py`, `repair*.py`, `propose_update.py`, `review*.py`). The duplicated `reasons_lib/` package implies either an in-progress rename or a shipped regression; that's the first thing to resolve.

## 3. Module Map

| Directory / Group | Responsibility |
|---|---|
| `reasons/cli.py`, `reasons/mcp_server.py` | **Entry points** — CLI dispatch and MCP tool server |
| `reasons/__init__.py`, `api.py` | **Public API** — package façade and orchestration |
| `reasons/storage.py`, `pg.py` | **Persistence** — SQLite backend, Postgres backend, propagation engine |
| `reasons/metadata.py` | **Shared model** — Node/Justification types, source metadata schema |
| `reasons/verify.py`, `merkle.py` | **Integrity** — invariants, consistency, content hashing |
| `reasons/contradictions.py`, `derive.py`, `repair*.py`, `propose_update.py` | **TMS reasoning** — nogoods, DDB, derivation, repair, updates |
| `reasons/network.py`, `hf.py` | **External I/O** — HTTP fetch with cache, HuggingFace integration |
| `reasons/import_*.py`, `export_*.py`, `build_wiki.py`, `check_stale.py`, `compact.py` | **Import/export & inspection** — round-trip formats, wiki build, staleness, token budgeting |
| `reasons/llm.py`, `ask.py`, `cluster.py` | **LLM/semantic** — LLM calls, ask interface, clustering (optional `llm`/`cluster` extras) |
| `reasons/mind_service.py`, `pubsub.py`, `report_*.py`, `review*.py`, `propose_update.py` | **Agent integration & review workflows** |
| `reasons_lib/` | **Duplicate of `reasons/`** — missing `merkle.py`; investigate divergence |
| `tests/` | ~65 test files mirroring the modules; the *fastest* map of responsibilities |
| `.github/workflows/tests.yml` | CI — confirms the canonical test entrypoint |
| `CLAUDE.md`, `README.md`, `CHANGELOG.md` | Docs & conventions |

## 4. Exploration Strategy

1. **Resolve the duplication first** — diff `reasons/` vs `reasons_lib/` to determine which is canonical and whether `reasons_lib` is dead weight or a divergent fork. This determines whether later findings should be applied once or twice.
2. **Orient with docs & config** — `README.md` (done), `CLAUDE.md`, `pyproject.toml`, `CHANGELOG.md`, `.github/workflows/tests.yml`. Establishes vocabulary (premise, justification, nogood, SL/CP, outlist) and the optional-dependency groupings.
3. **Follow the runner's path inward** — `reasons/__init__.py` → `reasons/cli.py:main` (dispatch table) → one command end-to-end (e.g. `reasons nogood`) tracing through `api.py` → `contradictions.py` → `storage.py`. This reveals the layering in a single concrete flow.
4. **Read the model next** — `metadata.py` (types) then `verify.py` (transition invariants). These encode the intended semantics more precisely than the README.
5. **Read the engine** — `storage.py` (propagation/backtracking) paired with `tests/conftest.py` + `tests/test_backtracking.py` + `tests/test_storage.py`. The tests will pin down what the code *actually* guarantees.
6. **Then the peripheral surfaces** — MCP server, import/export, network/HF, LLM/ask/cluster — because they're consumers of the core and less critical to understanding the system's semantics.

---

## Topics to Explore

- [file] `CLAUDE.md` — Project-specific conventions and workflow; fastest way to understand intended usage and refactoring rules.
- [file] `pyproject.toml` — Entry points, optional-dependency groups, and package list; reveals functional partitioning and flags the `reasons_lib` duplicate.
- [file] `reasons/__init__.py` — Package façade; what's exported here defines the intended public surface and points to the real core modules.
- [function] `reasons/cli.py:main` — The primary entry point and subcommand dispatch table; the single most efficient way to map capabilities to modules.
- [file] `reasons/api.py` — Orchestration layer between CLI/MCP commands and storage; the natural seam to read core operations end-to-end.
- [file] `reasons/storage.py` — Likely houses both persistence (SQLite) *and* the propagation engine (retraction cascade, restoration, log); the heart of the TMS.
- [file] `reasons/metadata.py` — Shared node/justification data model and source-metadata schema (types, access tags, hashes); dependency hub.
- [file] `reasons/verify.py` — Consistency invariants (dependents integrity, no dangling dependents); encodes the semantics the code is supposed to preserve.
- [file] `reasons/contradictions.py` — Nogood detection and dependency-directed backtracking; the non-trivial algorithm that justifies the whole project.
- [file] `reasons/derive.py` — Automatic (budgeted) derivation of new nodes; reveals how the graph grows programmatically.
- [function] `reasons/mcp_server.py:main` — Second entry point; the MCP tool set reveals which API ops are considered first-class for agent use.
- [file] `reasons/merkle.py` — Content-addressing/integrity; the module that distinguishes the two packages and likely underlies `check-stale`/`hash-sources`.
- [file] `tests/conftest.py` — Fixtures reveal the real DB bootstrap, node/justification construction, and test-time data model better than the README.
- [file] `reasons_lib/` — Duplicate package missing `merkle.py`; must be diffed against `reasons/` to determine if it is dead code or a divergent fork.
- [general] `propagation-vs-backtracking-boundary` — Where the retraction cascade ends and dependency-directed backtracking begins; the key conceptual seam to nail down when reading `storage.py` + `contradictions.py`.
