---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# File: CLAUDE.md

# `CLAUDE.md` — Explanation

> Note up front: this is **not executable code**. It is the project-memory / agent-context file that Claude Code (and any human contributor) loads when working in this repo. Its "contract" is informational, and its failure mode is **documentation drift**, not exceptions.

---

## 1. Purpose

`CLAUDE.md` at the repo root is the conventional Claude Code memory file: it is auto-loaded into context whenever an agent operates in this repository, and it doubles as the fast-path onboarding doc for a new engineer. It owns four responsibilities:

1. **Concept priming** — teaches the Doyle-1979 Truth Maintenance System vocabulary (Node, Premise, Justification, Outlist, Nogood, entrenchment) so that later instructions like "cascade the retraction" are unambiguous.
2. **Navigation map** — the `## Project Structure` block tells the reader which of the ~34 modules under `reasons/` does what, so they don't have to grep.
3. **Executable recipes** — the `## Commands` block is a copy-pasteable CLI manifest (`uv run reasons …`). For an agent this is effectively a tool schema in prose.
4. **Rationale record** — `## Design Decisions` explains *why* (SQLite over markdown, no STALE status, retaining retracted nodes, BFS propagation, the `reasons` vs `rms` naming), so future changes don't re-litigate settled trade-offs.

It explicitly does **not** own: the CLI implementation, the truth-value semantics, or the test suite — it only *describes* them. When the description and the code disagree, the code wins and this file is the bug.

---

## 2. Key Components

There are no classes or functions here; the "components" are the sections and the terminology they define. Treat each definition as a contract.

### `## What This Is`
Two claims with maintenance weight:
- The system implements **Doyle's 1979 TMS**.
- The name was changed `rms` → `reasons` because an **ablation showed a 5 pp accuracy delta** (LLMs confuse `rms` with root-mean-square). This number appears **twice** — here and again as the final bullet of `## Design Decisions`. If you ever re-run the ablation, both spots must change together.

### `## Key Concepts` — the semantic contract
| Term | Contract stated in the doc |
|---|---|
| **Node** | A belief with truth value `IN` or `OUT`. |
| **Premise** | A node with **no justifications**; `IN` by default. |
| **Justification** | `SL` = *all* antecedents must be `IN` **AND** all outlist entries must be `OUT`. |
| **Outlist** | The non-monotonic clause: "believe X unless Y is believed". |
| **Retraction cascade** | When a node goes `OUT`, dependents whose justifications become invalid also go `OUT`. |
| **Restoration** | When a retracted node returns `IN`, dependents are recomputed automatically. |
| **Nogood** | A set of nodes that cannot all be `IN` simultaneously. |
| **Backtracking** | On nogood detection, walk backward to the responsible premise using **entrenchment scoring**. |
| **Challenge/Defend** | Dialectical layer on top of propagation. |
| **Multiple justifications** | A node is `IN` if **ANY** justification is valid — i.e. OR-of-ANDs, not AND-of-ANDs. |
| **Summary nodes** | Abstract over groups of nodes in `compact` output. |

The `SL` + "ANY justification" pair is the single most important thing to internalise: it is what makes restoration cheap and what makes `explain`/`trace` well-defined.

### `## Project Structure`
A curated (deliberately incomplete) map. The doc calls out `reasons/` as the real package with `reasons_lib/` as a **backwards-compat shim that re-exports from `reasons/`**. Specific module claims worth verifying before you rely on them:
- `reasons/__init__.py` — holds the `Node`, `Justification`, `Nogood` dataclasses.
- `reasons/network.py` — the `Network` class: dependency graph + propagation + dialectical operations.
- `reasons/api.py` — functional API returning **plain dicts**, used by both the CLI and LangGraph.
- `reasons/check_stale.py` — source-hash comparison **plus `hash_sources`** (the backfill used by `hash-sources`).
- `reasons/merkle.py` — per-justification Merkle hashes for detecting text mutation.
- `reasons/compact.py` — token-budgeted summary with summary-node support.

### `## Commands`
~20 invocations, all prefixed `uv run reasons …`. This block encodes several implicit contracts:
- A **console script named `reasons`** must exist (i.e. `[project.scripts]` in `pyproject.toml`) and `uv run` must be able to resolve it — meaning the project is installed into the uv-managed venv.
- Command families you can infer: mutation (`init`, `add`, `retract`, `assert`, `propagate`), inspection (`status`, `show`, `explain`, `trace`, `log`, `search`, `list`), dialectical (`challenge`, `defend`, `nogood`), I/O (`import-beliefs`, `import-json`, `export`, `export-markdown`), integrity (`check-stale`, `hash-sources`, `check-integrity`, `backfill-hashes`), and presentation (`compact --budget N`).
- Flag semantics worth pinning: `--sl a,b` sets the support list, `--unless y` sets the outlist, **no flag ⇒ premise**, `--premises` filters `list`, `--budget 500` is in tokens.

### `## Running Tests`
`uv run --extra test pytest tests/ -v` asserts an **optional dependency group named `test`** exists in `pyproject.toml`. The trailing claim "**211 tests**" is a hard-coded count that silently rots; it is descriptive, not enforced.

### `## Design Decisions`
Six bullets, each a *rejected alternative* plus a reason. These are the guardrails against scope creep:
- SQLite over markdown ⇒ ACID during cascades.
- No `STALE` truth value ⇒ staleness is an *event* that triggers retraction, not a state.
- Retracted nodes are retained ⇒ enables restoration without rederivation (Doyle's insight).
- **BFS** propagation ⇒ every dependent is recomputed.
- Entrenchment scoring ⇒ evidence outranks speculation during backtracking.
- `reasons` over `rms` ⇒ the 5 pp ablation result.

---

## 3. Patterns

- **Agent-context file / project memory.** The root-level `CLAUDE.md` convention means this content is injected into an LLM's prompt. Everything here is therefore also *prompt engineering*: short, declarative, ordered concept-first. The "5pp" justification for the rename is only in the doc because the model needs to know not to reintroduce `rms` aliases.
- **Progressive disclosure.** Concepts → structure → commands → tests → rationale. A reader can stop after section 2 or 3 and still be productive; depth increases as you go down.
- **Rationale-first documentation.** Design decisions are recorded with the *reason* and the *rejected alternative*, not just the choice — the pattern that prevents future agents from "helpfully" switching to markdown storage.
- **Shim / re-export for backwards compatibility.** `reasons_lib/` mirrors the `reasons/` file layout so `import reasons_lib.X` keeps working after the rename. This is the classic compatibility-package pattern; it imposes a rule that **every new module in `reasons/` needs a counterpart in `reasons_lib/`**.
- **Docs as a curated subset.** The structure and command lists are intentionally partial (see §6) to keep the context window small — a deliberate trade of completeness for token budget.

---

## 4. Dependencies

**What this file depends on (conceptually — nothing at runtime):**
- `pyproject.toml` — supplies the `reasons` console script referenced by every `uv run reasons …` line, and the `test` extra used by the pytest command.
- `reasons/network.py`, `reasons/__init__.py` — the source of truth for all Key Concepts.
- `reasons/cli.py` — the source of truth for every command listed.
- `reasons/check_stale.py`, `reasons/merkle.py`, `reasons/compact.py` — back the integrity and summary commands.
- `tests/` — backs the "211 tests" claim.

**What depends on it:**
- Claude Code / any LLM agent operating in the repo (auto-loaded as memory).
- Human contributors as the entry document.
- `.claude/skills` (present in the tree) may layer additional instructions on top of it — worth checking whether skills duplicate or override anything here.
- Nothing imports it; it is not part of the Python dependency graph.

---

## 5. Flow

No control flow. The read path is linear and intentional:

1. **Orientation** (`What This Is`) — one paragraph, names the algorithm and the project.
2. **Vocabulary load** (`Key Concepts`) — must precede anything else, because `--sl`, `--unless`, `nogood`, and "cascade" are meaningless without it.
3. **Locate code** (`Project Structure`) — module → responsibility table, plus the shim warning.
4. **Act** (`Commands`) — the reader/agent picks a verb and runs it.
5. **Verify** (`Running Tests`) — a single canonical invocation, so there is no ambiguity about how to run the suite.
6. **Do not re-decide** (`Design Decisions`) — the reader is told which questions are already answered.

For an agent, the practical loop is: load concepts → choose a `uv run reasons …` command (or edit the module named in Project Structure) → run `uv run --extra test pytest tests/ -v` → check Design Decisions before proposing a structural change.

---

## 6. Invariants

Rules this document asserts. Breaking one of these without updating the doc creates drift; breaking one in the *code* is a semantic regression.

**Truth-value semantics**
- A node has exactly one of two truth values: `IN` or `OUT`. **There is no `STALE` state.** Staleness is detected and converted into a retraction.
- A node is `IN` iff **at least one** justification is satisfied (OR over justifications).
- A justification is satisfied iff **every** SL antecedent is `IN` **and every** outlist node is `OUT` (AND within a justification).
- A premise is a node with zero justifications and is `IN` by default.
- A nogood may never have all its members `IN` simultaneously.

**Lifecycle**
- Going `IN → OUT` must cascade to all invalidated dependents.
- Going `OUT → IN` must automatically recompute dependents (restoration).
- Retracted nodes are **never deleted** from the DB.
- Propagation is **BFS**, and it recomputes *all* dependents of a changed node — not just the first invalidated path.

**Packaging**
- `reasons` is the canonical import name; `reasons_lib` is a shim only and must not accumulate unique logic.
- `import-json` is **lossless**; `import-beliefs` (markdown) is not.
- Merkle hashes are per-justification; `check-integrity` detects text mutation, `backfill-hashes` fills gaps for legacy nodes.

**Documentation-level invariants**
- The `reasons` console script and `test` extra must exist in `pyproject.toml`, or every command in §Commands fails at invocation.
- The "211 tests" figure and the "5pp" figure are assertions the doc makes about the repo; both appear as unverified literals.

**Known drift to be aware of (verified against the tree provided):**
- `## Project Structure` omits modules that clearly exist: `reasons/derive.py`, `plan`-adjacent files, `cluster.py`, `contradictions.py`, `review*.py`, `repair*.py`, `report_*.py`, `pg.py`, `mcp_server.py`, `pubsub.py`, `mind_service.py`, `hf.py`, `export_card.py`, `build_wiki.py`, `ask.py`. The `## Commands` block similarly omits `derive`, `cluster`, `contradictions`, `mcp-server`, `pg`, `report-*`, `review-*`, `repair*`, `mind-service`, `export-card`, `build-wiki`, `ask`.
- The doc's `tests/` list is a small subset — `test_derive.py`, `test_pg.py`, `test_contradictions.py`, `test_review*.py`, `test_repair*.py`, `test_mcp_server.py`, etc. are absent from the description.
- `reasons/merkle.py` exists but the parallel `reasons_lib/` file listing has **no `merkle.py`** counterpart — if the shim is file-per-module, `import reasons_lib.merkle` may not resolve, contradicting the "re-exports from `reasons/`" claim.

So: treat the module/command/test lists as *illustrative*, and always confirm against the actual directory.

---

## 7. Error Handling

This file produces no errors and swallows none — but it defines the error semantics of the system it documents, and those definitions are the ones a maintainer will be held to:

- **Staleness is not an error state.** `check-stale` detects changed sources and the response is a **retraction**, not a `STALE` flag. Any code path that introduces a third truth value violates the documented model.
- **Contradiction is a first-class event.** `nogood A B` doesn't raise; it records the constraint and then *backtracks*, using entrenchment scores to nominate the responsible premise. Failure to resolve is expressed as "which premise to retract", not as an exception.
- **Integrity violations are surfaced, not fatal.** `check-integrity` compares stored Merkle hashes against recomputed ones and reports mutations; `backfill-hashes` exists precisely so that missing hashes are a recoverable condition rather than an error.
- **Import asymmetry is the documented safety boundary.** `import-json` round-trips losslessly (`test_import_json.py`, `test_roundtrip.py`), whereas `import-beliefs` from markdown is lossy by design — the doc does not promise you can recover everything through the markdown path.
- **Durability is delegated to SQLite.** The stated reason for choosing SQLite is **ACID transactions during propagation cascades**, i.e. a crash mid-cascade must not leave the network half-retracted. That is the error-handling strategy: atomicity rather than rollback logic in Python.
- **Test failures are the only enforcement mechanism.** Nothing validates that the doc matches the code; the invariants above are only backed by `tests/test_network.py`, `test_outlist.py`, `test_backtracking.py`, `test_dialectical.py`, `test_merkle.py`, etc.

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class is where every invariant in §6 is actually enforced (BFS propagation, cascade, restoration, `challenge`/`defend`, nogood backtracking); read it to confirm the doc's "BFS" and "ANY justification" claims.
- [file] `pyproject.toml` — Verifies the two implicit contracts of the `## Commands` block: the `reasons` console script and the optional `test` dependency group used by `uv run --extra test`.
- [file] `reasons/merkle.py` — Backs `check-integrity` and `backfill-hashes`; also the module apparently missing from the `reasons_lib/` shim, so it is the best probe of whether the shim is really a one-line re-export.
- [function] `reasons/check_stale.py:hash_sources` — The named backfill routine behind `hash-sources`, and the entry point for the "staleness triggers retraction, not a STALE status" decision.
- [general] `claude-md-drift-audit` — Diff the `## Project Structure`, `## Commands`, and "211 tests" claims against the real tree; the Provided listing already shows ~15 undocumented modules, which is the highest-value maintenance task here.

---

## Beliefs

- `justification-sl-requires-all-antecedents-in-and-all-outlist-out` — A node is IN only if at least one justification has every SL antecedent IN and every outlist node OUT.
- `reasons-lib-is-a-shim-for-reasons-package` — `reasons_lib/` exists solely as a backwards-compatibility re-export layer over `reasons/`, so new modules must be mirrored there.
- `retained-retracted-nodes-enables-restoration` — Retracted nodes stay in the SQLite DB specifically so that restoration can recompute dependents without rederivation.
- `no-stale-status-prevents-third-truth-value` — The system deliberately has only IN/OUT truth values; source staleness triggers a retraction instead of introducing a STALE state.
- `sqlite-is-required-for-atomic-propagation-cascades` — SQLite (not markdown) was chosen to provide ACID transactions during multi-node retraction cascades.
- `claude-md-lists-a-curated-subset-of-modules-and-commands` — The Project Structure and Commands sections document roughly half of the ~34 modules in `reasons/`, so both lists must be verified against the tree rather than trusted as complete.
