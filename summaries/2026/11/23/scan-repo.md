---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Scan: repo

# Reconnaissance Report: `ftl-reasons`

## 1. Critical Files

| # | File | Why it matters |
|---|------|----------------|
| 1 | `README.md` | The domain contract. Defines *node*, *justification* (SL/CP), *nogood*, *outlist*, *retraction cascade*, *dependency-directed backtracking*. Everything else implements these concepts. |
| 2 | `pyproject.toml` | Reveals packaging: **two packages are shipped** (`reasons` + `reasons_lib`), two entry points (`reasons.cli:main`, `reasons.mcp_server:main`), and a layered optional-dependency model (`pg`, `cluster`, `mcp`, `llm`, `vertex`, `langfuse`, `openai`, `api`). Zero required runtime deps — everything is opt-in. |
| 3 | `reasons/__init__.py` **and** `reasons_lib/__init__.py` | **The single biggest structural question.** The two package trees are near-identical (34 vs 33 files; `reasons_lib` lacks `merkle.py`). One is likely canonical, the other vendored/generated/legacy. Resolve this before reading anything else. |
| 4 | `reasons/storage.py` | Likely home of the data model: `Node`, `Justification`, `Nogood`, and the SQLite schema. With no separate `models.py`/`types.py`, shared types probably live here (or are split into `metadata.py` + `merkle.py`). |
| 5 | `reasons/network.py` | Despite the name, this is almost certainly the **belief dependency graph** (not HTTP). Core to understanding propagation: antecedent→consequent edges, dependent lookup. |
| 6 | `reasons/derive.py` | The TMS engine: truth-value recomputation, retraction cascade, restoration. Everything from `assert`, `retract`, `propagate` funnels through here. **Highest-priority logic file.** |
| 7 | `reasons/contradictions.py` | Nogood handling + dependency-directed backtracking — the feature that distinguishes this from a plain constraint system. The README's hardest-to-understand concept (minimal-disruption premise retraction) lives here. |
| 8 | `reasons/cli.py` | Primary entry point. The full command surface (~30 commands) and how each command wires `storage → network → derive → contradictions`. Read it to see the intended call graph. |
| 9 | `reasons/api.py` | Programmatic facade. Reveals the "blessed" public API surface that `mcp_server.py`, `ask.py`, and LLM tooling call into. |
| 10 | `reasons/mcp_server.py` | Second entry point; models the MCP tool contract. Tells you which operations are exposed to agents, i.e. what the maintainers consider the important operations. |
| 11 | `tests/conftest.py` | Fixtures encode the expected data model and setup invariants more concretely than docs. Fastest way to learn the object graph. |
| 12 | `tests/test_derive.py` + `tests/test_backtracking.py` + `tests/test_storage.py` | Behavioral specification for the three core mechanisms (propagation, DDB, persistence). Read as executable documentation. |
| 13 | `CHANGELOG.md` | At v0.58.0 with this many modules, history reveals which abstractions are stable vs. recently churned, and likely explains the `reasons`/`reasons_lib` split. |
| 14 | `CLAUDE.md` + `.claude/skills/reasons` + `.forge/*.json` | Meta-layer: this repo appears to be agent-managed (skill definitions, "forge" proposal/report artifacts, `costs/` ledger). Explains repo conventions before you "fix" anything. |

## 2. Architecture Sketch

This is a **Truth Maintenance System library** built around a persistent belief graph: nodes carry IN/OUT truth values, justifications encode support (SL: all antecedents IN; CP: assumptions consistent; outlist: non-monotonic "unless" defaults), and nogoods drive dependency-directed backtracking. A propagation engine (`derive.py`) recomputes truth values and cascades retraction/restoration, while `storage.py`+`network.py` provide persistence (SQLite default, optional Postgres) and graph traversal. Two entry points — a rich CLI (`reasons.cli:main`) and an MCP server (`reasons.mcp_server:main`) — sit over a programmatic facade (`api.py`), and an **LLM/agent layer** (`llm.py`, `ask.py`, `review*.py`, `repair*.py`, `propose_update.py`) uses the belief graph to review, contradict-check, and propose updates. Import/export bridges (`import_beliefs.py`, `export_markdown.py`, `build_wiki.py`) keep a dual currency: the SQLite DB and a markdown "beliefs registry." The glaring architectural twist is **two near-identical package trees** (`reasons/` and `reasons_lib/`), which must be disambiguated early — it likely reflects vendoring, a migration, or a distribution split rather than intentional design.

## 3. Module Map

**Core TMS engine (the heart)**
`storage.py`, `network.py`, `derive.py`, `verify.py`, `contradictions.py`, `metadata.py`, `merkle.py`, `compact.py`

**Entry points & orchestration**
`cli.py`, `api.py`, `ask.py`, `mcp_server.py`, `mcp_client.py`

**Persistence & sync backends**
`storage.py` (SQLite), `pg.py` (Postgres), `hf.py` (HuggingFace), `pubsub.py`, `mind_service.py`

**LLM / agent layer**
`llm.py`, `api.py` (LLM providers), `import_agent.py`, `propose_update.py`, `mind_service.py`

**Belief hygiene (review & repair)**
`review.py`, `review_justifications.py`, `review_premises.py`, `repair.py`, `repair_premises.py`, `contradictions.py`, `check_stale.py`, `cluster.py`

**Reporting**
`report_belief.py`, `report_gated.py`, `export_card.py`

**Import / export**
`import_beliefs.py`, `export_markdown.py`, `build_wiki.py`

**Meta / tooling**
`.forge/`, `.claude/skills/reasons`, `costs/`, `CLAUDE.md`, `.github/workflows/tests.yml`, `CHANGELOG.md`

**Tests** — 60+ files mirroring the modules; `conftest.py` + `test_derive/test_backtracking/test_storage` are the semantic spec.

## 4. Exploration Strategy

1. **Orient:** `README.md` → `pyproject.toml` → `CHANGELOG.md` (build mental model of features + deps + history).
2. **Resolve the duality:** skim `reasons/__init__.py` and `reasons_lib/__init__.py`, then `diff -r reasons/ reasons_lib/` on a couple of core files (`storage.py`, `derive.py`). Decide which tree is canonical *before* reading deeper.
3. **Learn the data model:** `storage.py` (+ `metadata.py`, `merkle.py`) — what a node/justification/nogood physically *is*.
4. **Learn the graph:** `network.py` — how dependents and antecedents are traversed.
5. **Learn the algorithm:** `derive.py` (propagation/retraction/restoration) then `contradictions.py` (nogoods + backtracking) and `verify.py` (invariants).
6. **See the wiring:** `cli.py` and `api.py` — map commands to engine calls.
7. **Validate with tests:** `tests/conftest.py`, `tests/test_derive.py`, `tests/test_backtracking.py`, `tests/test_storage.py` — confirm and stress the mental model.
8. **Move outward:** `mcp_server.py` (agent surface) → `llm.py`/`review*.py`/`repair*.py` (automation layer) → `import_beliefs.py`/`export_markdown.py` (durable formats) → `pg.py`/`pubsub.py`/`mind_service.py` (distributed backends).

## Topics to Explore

- [general] `package-duality` — Why `reasons/` and `reasons_lib/` are near-identical, which is canonical, and how imports cross between them (determines everything downstream).
- [file] `README.md` — Authoritative domain vocabulary (node, SL/CP justification, outlist, nogood, cascade, DDB) that all code must be read against.
- [file] `pyproject.toml` — Package declaration, entry points, and the optional-dependency layering that reveals intended module boundaries.
- [file] `reasons/__init__.py` — Public import surface; resolves whether `reasons` is a wrapper over `reasons_lib` or the real implementation.
- [file] `reasons/storage.py` — Shared data types and SQLite schema; the canonical object model.
- [file] `reasons/network.py` — Belief dependency graph: antecedent/dependent edges used by all propagation.
- [file] `reasons/derive.py` — The TMS propagation engine: assert, retract, cascade, restore.
- [function] `reasons/derive.py:propagate` — Core truth-value recomputation invoked by most CLI commands; the algorithmic center (name may be `recompute`/`propagate`).
- [file] `reasons/contradictions.py` — Nogood recording and dependency-directed backtracking to a single responsible premise.
- [file] `reasons/cli.py` — Command dispatch mapping ~30 user commands onto storage/derive/contradictions; best overall call graph.
- [file] `reasons/api.py` — Programmatic facade that MCP server and LLM tooling call; the "supported" API boundary.
- [file] `tests/conftest.py` — Test fixtures that concretely encode the expected object graph and DB lifecycle.
- [file] `tests/test_derive.py` — Executable specification of retraction cascades, restoration, diamond dependencies, multiple justifications.
- [file] `tests/test_backtracking.py` — Specification of nogood-driven backtracking behavior and minimal-disruption guarantees.
- [file] `reasons/mcp_server.py` — Agent-facing tool contract; shows which operations the maintainers consider essential.
