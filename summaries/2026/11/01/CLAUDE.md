---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: CLAUDE.md

# `CLAUDE.md` — Project Instruction File

## 1. Purpose

This is **not executable code** — it's an AI-assistant instruction file (the `CLAUDE.md` convention used by Claude Code and similar tools). Its sole responsibility is to give an LLM-based coding agent enough context to navigate, modify, and test this project *correctly* on the first pass. It functions as a compressed onboarding document that replaces what a human would get from reading `README.md`, architecture docs, and asking a senior dev questions.

Specifically it owns:
- **Domain vocabulary**: Locking in the precise meaning of terms like *Node*, *Justification*, *Outlist*, *Nogood*, *Entrenchment* so the AI doesn't conflate them.
- **Naming rationale**: Explaining *why* the project is called "reasons" and not "rms" (the 5pp ablation finding). Without this, an AI might "helpfully" rename things back.
- **Command surface**: The full CLI contract so the AI generates correct `uv run reasons …` invocations.
- **Design invariants**: The "why" behind choices (SQLite over markdown, IN/OUT over IN/OUT/STALE, retracted nodes persisting) that prevent the AI from "improving" them away.

A developer maintaining this file should treat it as a **contract with every AI session**. If you add a feature, rename a module, or change a design decision, you must update this file in the same PR — otherwise every future AI session will work from stale assumptions.

## 2. Key Components

Since this is a markdown document, the "components" are its sections and the assertions they make:

| Section | What it pins down | Maintenance risk if stale |
|---|---|---|
| **What This Is** | Project identity + the `rms` → `reasons` rename rationale | AI renames modules back to `rms` |
| **Key Concepts** | 10 domain terms with precise definitions (SL, OUTLIST, NOGOOD, BACKTRACKING, etc.) | AI generates incorrect propagation logic |
| **Project Structure** | Module map for `reasons/`, `reasons_lib/`, `tests/` | AI edits the wrong package (lib vs. reasons) |
| **Commands** | 25+ CLI subcommands with exact flags | AI produces invalid CLI calls in generated scripts |
| **Running Tests** | `uv run --extra test pytest tests/ -v`, 211 tests | AI uses wrong test invocation |
| **Design Decisions** | 6 "why" decisions (SQLite, no STALE, retracted-persist, BFS, entrenchment, naming) | AI "refactors" SQLite to JSON, adds STALE state, etc. |

### Critical invariants encoded here

- **`reasons/` is canonical; `reasons_lib/` is a backwards-compat shim.** The file explicitly says `reasons_lib/` "re-exports from `reasons/`." Any code change must go in `reasons/`, never `reasons_lib/`.
- **Two-valued logic only.** "No STALE status — uses IN/OUT only; staleness triggers retraction." This is a hard constraint on the state machine.
- **BFS propagation.** "BFS propagation ensures all dependents are recomputed when a node changes." The AI must not "optimize" this to DFS or lazy evaluation without understanding the cascade semantics.
- **Retracted nodes persist.** "Retracted nodes stay in DB (Doyle's insight: enables restoration without rederivation)." Deleting OUT nodes from storage would break `assert`/restoration.

## 3. Patterns

- **AI-prompt-as-code**: The file is structured as a prompt engineering artifact. The bolded key concepts, the fenced code blocks for commands, and the "Design Decisions" section are all deliberate choices to maximize LLM comprehension. It mirrors how you'd write system prompts for an agent.
- **Backwards-compat dual-package**: `reasons/` (canonical) + `reasons_lib/` (shim). The file makes this explicit so the AI doesn't "deduplicate" them.
- **Doyle's TMS as the conceptual anchor**: Every concept maps back to the 1979 paper. The file names the paper, the author, and the year — this is the intellectual root that all design decisions trace back to.
- **Ablation-study-driven naming**: The `rms` → `reasons` rename isn't arbitrary; it's grounded in a measured 5pp LLM accuracy delta. This sets a precedent that naming choices in this project should be evidence-based.
- **Command-table documentation**: The CLI commands are listed with exact flags (`--sl a,b`, `--unless y`, `--budget 500`) rather than described prose-ly. This is the "contract over narrative" pattern.

## 4. Dependencies

**This file imports nothing.** It is a leaf document. However, its *referential* dependencies are heavy:

**References out (must stay in sync with):**
- `reasons/` module structure — every file listed under "Project Structure" must exist
- `reasons/cli.py` — the Commands section must match the actual argparse/click definitions
- `tests/` — the "211 tests" count and the listed test files must be accurate
- `pyproject.toml` — the `uv run` commands and `--extra test` imply specific package metadata
- `reasons/network.py` — the Key Concepts section describes its API surface
- `reasons/storage.py` — the SQLite design decision
- `reasons/merkle.py` — the `check-integrity` and `backfill-hashes` commands
- `reasons/compact.py` — the `compact --budget` command

**Referenced by (imported by):**
- Every Claude Code / AI-assistant session that loads this repo
- Potentially `.github/workflows/` if CI injects this file as context for AI-based code review
- `.claude/skills/` directory (visible in repo tree) which may load this as part of skill definitions

**The file itself has no runtime dependencies**, but it is *semantically* coupled to ~15 source files. A rename in `reasons/` without updating `CLAUDE.md` creates a silent drift.

## 5. Flow

There is no execution flow — this is a static document. But the *information flow* for a reader (human or AI) is:

```
"What This Is"          →  I know what I'm in
  ↓
"Key Concepts"          →  I have the vocabulary
  ↓
"Project Structure"     →  I know where to look
  ↓
"Commands"              →  I know how to interact
  ↓
"Running Tests"         →  I know how to verify
  ↓
"Design Decisions"      →  I know what NOT to change
```

This is a deliberate top-down funnel: identity → semantics → location → interaction → verification → constraints. A developer adding a new section should respect this ordering.

## 6. Invariants

The file enforces (by documentation) several rules that the *code* enforces by implementation:

1. **Binary truth values.** Nodes are IN or OUT. There is no third state. The file explicitly says "No STALE status."
2. **Monotonic + non-monotonic justifications coexist.** SL (all antecedents IN) and OUTLIST (all outlist OUT) must *both* hold for a justification to be valid. This is stated as: "SL = all antecedents must be IN AND all outlist must be OUT."
3. **Any-justification semantics.** "A node is IN if ANY justification is valid." This is disjunctive, not conjunctive, across justifications.
4. **Cascade completeness.** "When a node goes OUT, all dependents whose justifications become invalid also go OUT." The propagation must be exhaustive (hence BFS).
5. **Restoration symmetry.** "When a retracted node comes back IN, dependents are automatically recomputed." The system is not only non-monotonic in retraction but also *restorable*.
6. **Entrenchment ordering in backtracking.** "Entrenchment scoring protects evidence over speculation." When a nogood is found, premises with higher entrenchment are less likely to be retracted.
7. **Merkle integrity.** Per-justification Merkle hashes exist to detect text mutations. `check-integrity` verifies them.

## 7. Error Handling

Since this is a documentation file, "error handling" means **what happens when the file is wrong or stale**:

- **Silent drift is the primary failure mode.** If a module is renamed, a command is added, or a design decision is reversed, and `CLAUDE.md` isn't updated, every subsequent AI session will generate code that references non-existent modules, uses removed flags, or violates design invariants. There is no compiler to catch this.
- **The "211 tests" count** is a hard-coded number. It will become wrong the moment a test is added or removed. This is a maintenance trap — consider whether to keep the exact count or just say "comprehensive test suite."
- **The `reasons_lib/` shim** is described as "re-exports from `reasons/`." If someone adds a module to `reasons/` but forgets `reasons_lib/`, the shim breaks silently and the file's implication that they're interchangeable becomes false.
- **No versioning or date.** The file has no "last updated" marker. A contributor has no way to know if the Design Decisions section reflects the current state or was written six months ago before a major refactor.

---

## Topics to Explore

- [file] `reasons/network.py` — The core propagation engine (BFS cascade, justification validation, dialectical challenge/defend). Understanding this is prerequisite to understanding why the CLAUDE.md invariants exist.
- [file] `reasons/cli.py` — The command surface that the Commands section of CLAUDE.md documents. Verify every listed subcommand and flag actually exists; this is where drift is most likely.
- [file] `reasons/merkle.py` — Implements per-justification Merkle hash integrity. The CLAUDE.md mentions `check-integrity` and `backfill-hashes` commands but doesn't explain the hash structure.
- [general] `doyle-tms-1979` — The foundational paper (Doyle, J., "A Truth Maintenance System," 1979) that this system implements. Reading it clarifies *why* the IN/OUT binary, retracted-node persistence, and justification structure are designed the way they are.
- [file] `reasons_lib/__init__.py` — The backwards-compat shim. Verify it truly re-exports everything from `reasons/` and nothing has drifted. This is the single point of failure for the dual-package design.

---

## Beliefs

- `claudemd-documents-binary-truth-model` — The system supports exactly two truth values (IN, OUT); there is no STALE, UNKNOWN, or PARTIAL state, and staleness is handled by triggering retraction rather than introducing a third value.
- `reasons-lib-is-shim-for-reasons` — `reasons_lib/` is a backwards-compatibility alias that re-exports from `reasons/`; all new code must be written in `reasons/`, and `reasons_lib/` must never contain independent logic.
- `justification-validity-requires-both-sl-and-outlist` — A single justification is valid only when ALL antecedents (SL) are IN **and** ALL outlist nodes are OUT; multiple justifications on the same node are disjunctive (ANY valid justification keeps the node IN).
- `retracted-nodes-persist-in-storage` — Nodes marked OUT are never deleted from the database; they remain queryable and restorable, which is the mechanism that enables automatic dependent recomputation on restoration.
- `claudemd-must-update-with-code` — The file contains hardcoded counts (211 tests), module lists, and command tables that have no runtime enforcement; any code change that adds, removes, or renames a module/command/test must update CLAUDE.md in the same change or AI sessions will generate incorrect code.
