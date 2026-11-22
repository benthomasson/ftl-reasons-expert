---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Scan: repo

# Reconnaissance: `ftl-reasons` (Truth Maintenance System)

## 1. Critical Files

**`reasons/cli.py`**
The primary entry point (`reasons = "reasons.cli:main"`). It maps every TMS operation (`add`, `retract`, `assert`, `explain`, `nogood`, `challenge`, `defend`, `trace`, `check-stale`, `compact`, …) onto the underlying network/storage layer. Reading it first gives you the full external vocabulary of the system and shows how thin the business logic is relative to the core engine.

**`reasons_lib/__init__.py` and `reasons/__init__.py`** (both)
There are *two* nearly identical packages. `reasons_lib/` is missing `merkle.py`, and `pyproject.toml` ships both. This is the single biggest ambiguity in the repo: is `reasons_lib` a vendored/legacy copy, or a library-only split where `reasons/` adds merkle + CLI? The `__init__.py` exports and any docstring here answer that question and determine which tree is canonical for everything else.

**`reasons/storage.py`**
The persistence layer (SQLite by default, with `pg.py` as the Postgres alternative). It defines the actual data model — nodes, justifications (SL/CP, inlist/outlist), nogoods, propagation log, source hashes. The schema *is* the domain model; everything else is a transformation over these tables. Study this before `network.py`.

**`reasons/network.py`**
The heart of the TMS: truth-value computation, retraction cascade, restoration, diamond dependencies, multiple-justification resolution. All the behavior the README advertises (cascades, restoration without rederivation, outlist defaults) lives here. This is the file to understand most deeply.

**`reasons/verify.py`**
Justification validity logic — the predicate that decides whether an SL/CP justification holds given current IN/OUT states. It's the small, testable core that `network.py` calls to do propagation. Pin this down and the cascade algorithm becomes obvious.

**`reasons/contradictions.py`** + **`reasons/repair.py` / `repair_premises.py`**
Nogood handling and dependency-directed backtracking. This is the intellectually hardest part of Doyle's TMS — tracing backward to retract the *responsible premise* with minimal disruption. `contradictions.py` detects/holds nogoods; `repair*.py` implements the backtracking/repair strategies.

**`reasons/merkle.py`** (present **only** in `reasons/`)
Content-addressing / integrity for nodes and sources (`hash-sources`, `check-stale`, `test_merkle.py`). Its one-sided presence is a strong hint that `reasons/` is the newer canonical package. Also central to sync/import features.

**`reasons/pg.py`**
The Postgres backend — the alternative to SQLite storage. Worth reading after `storage.py` because it reveals which parts of the data model are abstracted vs. SQLite-specific, and it's the boundary for `test_pg.py` / `test_pg_dispatch.py`.

**`reasons/mcp_server.py`**
The second living entry point (`reasons-mcp = "reasons.mcp_server:main"`). Exposes the TMS over the Model Context Protocol. It's the best example of the system being consumed as a *service* rather than a CLI, and it pairs with `mcp_client.py`.

**`tests/conftest.py`**
Fixtures for the TMS. In a 70-file test suite, conftest reveals how databases/networks are constructed, which is more reliable than reading the production constructor paths.

**`tests/test_backtracking.py`, `tests/test_outlist.py`, `tests/test_derive.py`, `tests/test_nogood_id.py`**
The behavioral specification of the hard parts: dependency-directed backtracking, non-monotonic default reasoning (outlist), derivation, and nogood identity. Because the README only narrates these behaviors, these tests are the ground truth for edge cases.

**`CLAUDE.md`** and **`README.md`**
README gives the product-level mental model; `CLAUDE.md` typically carries repo-specific conventions, package-split notes, and contributor guidance — exactly the context missing from the directory listing. Read `CLAUDE.md` early to disambiguate the two-package situation.

**`.claude/skills/reasons/`** and **`.forge/report.json` / `.forge/proposed-entries.json`**
Agent/tooling metadata describing how the repo is meant to be navigated (skills) and what "forge" thinks the notable entries are. Low-risk, high-signal for understanding author intent.

---

## 2. Architecture Sketch

`ftl-reasons` is a **layered Python implementation of Doyle's (1979) Truth Maintenance System**, where a relational persistence layer (`storage.py` over SQLite, `pg.py` over Postgres) stores nodes, justifications, nogoods, and a propagation log, and a graph engine (`network.py` + `verify.py`) recomputes IN/OUT truth values, propagates retraction cascades, restores dependents, and performs dependency-directed backtracking via `contradictions.py`/`repair*.py`. The **CLI (`cli.py`) is the primary surface**, with an **MCP server** as a secondary programmatic surface and `mind_service.py`/`pubsub.py`/`propose_update.py`/`review*.py` forming a higher-level service-and-review layer. **Optional extras gate capabilities**: LLM/LangChain (`llm.py`, `api.py`, `ask.py`, `derive.py`, `compact.py`), embeddings/clustering (`cluster.py`, `hf.py`), and Postgres. A suspicious **duplicate `reasons/` vs `reasons_lib/` package split** (with `merkle.py` existing only in `reasons/`) indicates either a migration/vendoring situation or a CLI-vs-library split that must be resolved before any broad refactor.

---

## 3. Module Map

| Group | Files | Responsibility |
|---|---|---|
| **Core TMS engine** | `network.py`, `verify.py`, `derive.py`, `contradictions.py`, `repair.py`, `repair_premises.py`, `merkle.py` | Propagation, justification validity, derivation, nogoods, backtracking/repair, content hashing |
| **Persistence** | `storage.py`, `pg.py` | SQLite default store; Postgres backend |
| **CLI / public surfaces** | `cli.py`, `mcp_server.py`, `mcp_client.py`, `api.py`, `mind_service.py`, `pubsub.py` | Command entry points, MCP exposure, service layer, events |
| **Import/export & sync** | `import_agent.py`, `import_beliefs.py`, `export_card.py`, `export_markdown.py`, `build_wiki.py`, `hf.py`, `merkle.py` | Round-tripping beliefs, wiki/card export, Hub/remote sync |
| **LLM-assisted workflows** | `llm.py`, `ask.py`, `derive.py`, `compact.py`, `cluster.py`, `propose_update.py`, `review*.py`, `report_*.py` | Query, summarization, clustering, draft-and-review pipelines |
| **Maintenance / hygiene** | `check_stale.py`, `metadata.py`, `compact.py`, `repair*.py` | Source-staleness detection, metadata, token-budget summaries |
| **Tests** | `tests/` (70+ files) | Behavioral spec for engine + each surface |
| **Repo meta** | `CLAUDE.md`, `README.md`, `pyproject.toml`, `.forge/`, `.claude/skills/`, `.github/workflows/tests.yml` | Conventions, config, agent skills, CI |

---

## 4. Exploration Strategy

1. **Resolve the duplicate-package question first** — `CLAUDE.md`, then `reasons/__init__.py` vs `reasons_lib/__init__.py`, then `pyproject.toml`. Do not read engine code until you know which tree is authoritative.
2. **Data model** — `storage.py` (schema, justification/nogood tables). This anchors every concept in the README.
3. **Core algorithm** — `verify.py` → `network.py`. Read in that order: validity predicate first, then the propagation/cascade loop that applies it.
4. **Hard reasoning** — `contradictions.py` → `repair*.py`, cross-checked against `test_backtracking.py`, `test_outlist.py`, `test_nogood_id.py`.
5. **Wiring & interfaces** — `cli.py`, then `mcp_server.py`. Watch how little logic they add; everything routes to the engine.
6. **Alternate backends & integrity** — `merkle.py`, `pg.py`.
7. **Feature layer outwards** — `import_beliefs.py`/`export_markdown.py` (round-trip), `check_stale.py`, `llm.py`/`ask.py`/`compact.py`, then the service/review cluster (`mind_service.py`, `propose_update.py`, `review*.py`).
8. **Use conftest fixtures** as the calibration point whenever a constructor signature surprises you.

---

## Topics to Explore

- [file] `CLAUDE.md` — Repo conventions and (likely) the answer to why `reasons/` and `reasons_lib/` both exist; read before any code.
- [file] `reasons_lib/__init__.py` — Together with `reasons/__init__.py`, defines the public API and clarifies the two-package split.
- [file] `reasons/__init__.py` — Canonical (or not) package exports; compare directly against `reasons_lib/__init__.py` to confirm which tree is authoritative.
- [file] `reasons/storage.py` — The schema *is* the domain model: nodes, SL/CP justifications, inlist/outlist, nogoods, propagation log, source hashes.
- [function] `reasons/verify.py:is_justification_valid` (or equivalent) — Smallest unit of TMS semantics: given IN/OUT states and inlist/outlist, does a justification hold?
- [file] `reasons/network.py` — The engine: truth computation, retraction cascade, restoration-without-rederivation, diamond dependencies.
- [file] `reasons/contradictions.py` — Nogood recording and the entry point to dependency-directed backtracking.
- [file] `reasons/repair.py` — The "retract the responsible premise with minimal disruption" algorithm; hardest reasoning in the codebase.
- [file] `reasons/merkle.py` — Present only in `reasons/`; content hashing for sources/nodes and the strongest signal about package roles.
- [file] `reasons/cli.py` — Primary entry point; enumerates every user-facing operation and how it maps onto engine calls.
- [file] `reasons/mcp_server.py` — Secondary entry point; shows the TMS exposed as a service rather than a CLI.
- [file] `reasons/pg.py` — Postgres backend; reveals the storage abstraction boundary and what's SQLite-specific.
- [file] `tests/conftest.py` — Fixtures define how a `Network`/store is constructed and seeded — the fastest path to the real data model.
- [file] `tests/test_backtracking.py` — Ground-truth behavioral spec for dependency-directed backtracking, the README's headline feature.
- [file] `tests/test_outlist.py` — Pins non-monotonic (default/`--unless`) semantics and dialectical challenge/defend behavior.
- [general] `truth-maintenance-core` — The Doyle model (IN/OUT, SL vs CP justifications, cascade, restoration) as the conceptual spine every file serves.
