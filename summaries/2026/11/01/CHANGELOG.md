---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: CHANGELOG.md

# CHANGELOG.md — Project History & Release Notes

## Purpose

This file is the **human-readable release history** for the `ftl-reasons` package. It serves three distinct roles:

1. **Release audit trail** — A chronological (reverse-chronological, newest-first) record of every shipped feature, fix, and breaking change, tied to issue/PR numbers for traceability.
2. **Onboarding map** — A new contributor can read top-to-bottom and reconstruct *why* the architecture looks the way it does (e.g., v0.3.0 explains the rename from `rms` → `reasons` based on an ablation study).
3. **Consumer contract documentation** — Downstream users (agents, CI pipelines, other repos) can diff changelog entries to understand what a version bump actually changes at the CLI/API surface level.

It does **not** replace `README.md` (which is a "what is this and how do I use it" doc) or inline docstrings. It answers *"what changed between 0.14 and 0.18, and why should I care?"*

## Key Components

This is a static Markdown document, so there are no functions or classes. The meaningful "components" are the **version blocks** and their sub-sections:

| Version | Headline Change | Why It Matters to a Maintainer |
|---------|----------------|-------------------------------|
| **0.18.0** | Access tags / RBAC | Largest surface-area change: touches every read command, adds a new `PermissionError` path, changes export semantics (nogood filtering), and introduces tag inheritance/propagation rules. This is the version where the data model gets a second dimension beyond truth-values. |
| **0.17.0** | `add-justification`, `--any` (OR-mode), `sync-agent` | Shifts the justification model from purely conjunctive (AND) to optionally disjunctive (OR). The `sync-agent` command implies a "remote-wins" reconciliation policy that interacts with `import-agent`'s namespace prefixing. |
| **0.16.0** | Import-agent outlist fix, `_retracted` flag fix | Two correctness bugs in the propagation engine. The `_retracted` fix is subtle: the flag must be set even when the node is *already* OUT, meaning the flag is an **idempotent audit marker**, not a state transition. |
| **0.14.0** | `recompute_all` fixpoint with iteration cap | This reveals the propagation engine's termination strategy: bounded iteration rather than unbounded fixpoint. A maintainer needs to know the cap exists to avoid infinite loops on cyclic graphs. |
| **0.13.0** | CI workflow, cascade output split | The "Went OUT / Went IN" split is a UX contract: `retract` and `assert` commands produce two distinct sections. Tests in `tests/test_dialectical.py` and `tests/test_backtracking.py` likely assert on this format. |
| **0.12.0** | `deduplicate` with Jaccard similarity | Introduces a clustering algorithm into the belief network. The "review-then-accept" workflow means deduplication is **not** atomic — it's a two-phase commit. |
| **0.10.0 → 0.7.0** | `derive`, `import-agent`, `what-if` | The "LLM reasoning layer" is added incrementally. `derive` proposes new beliefs via LLM; `import-agent` brings in multi-agent beliefs; `what-if` is a dry-run simulation. These three form the "reasoning" triad on top of the core TMS. |
| **0.1.0** | Core TMS | The foundation: SL justifications, outlist (non-monotonic reasoning), nogood detection, dependency-directed backtracking, SQLite persistence. |

## Patterns

1. **Keep a Changelog 1.1.0 format** — Strict categorization into `Added`, `Changed`, `Fixed`, `Removed`, `Security`. The file never uses `Removed` or `Security` (no breaking removals, no CVEs yet).

2. **Semantic Versioning (SemVer)** — Only the minor version increments (0.1.0 → 0.18.0). No patch releases, no major bump. This is consistent with a pre-1.0 project where the API is still evolving but the project has never broken a major contract.

3. **Issue/PR cross-referencing** — Entries like `(#38, PR #42)` or `(#16, PR #17)` create a bidirectional link to the VCS. A maintainer can jump from changelog → PR → code diff.

4. **Reverse-chronological ordering** — Newest version at top. This is the Keep a Changelog convention and makes the file naturally "readable" without scrolling.

5. **CLI-centric language** — Changes are described in terms of *commands* (`add`, `derive`, `retract`) and *flags* (`--any`, `--min-depth`, `--visible-to`), not internal function names. This is intentional: the changelog is written for the *user of the CLI*, not the reader of `reasons/api.py`.

6. **Dual-package structure** — The repo has both `reasons/` and `reasons_lib/` (identical file sets). The changelog doesn't call this out, but a maintainer should know that every feature listed applies to *both* packages. The `reasons_lib` variant is the importable library; `reasons` is the CLI entry point.

## Dependencies

**What this file depends on (upstream):**
- `pyproject.toml` — The version number in `pyproject.toml` must match the latest entry here. A mismatch is a release-integrity bug.
- `.github/workflows/` — CI (added in 0.13.0) enforces that tests pass; the changelog's "Fixed" entries are only valid if the corresponding test in `tests/` passes.
- Issue tracker (GitHub) — The `(#N, PR #M)` references are only meaningful if those issues/PRs exist.

**What depends on this file (downstream):**
- **Release automation** (if any) — A publishing script might parse the top `## [X.Y.Z]` header to determine the version to publish to PyPI.
- **Downstream consumers** — Agents and other tools that read the changelog to decide whether to upgrade.
- **`CLAUDE.md`** — The project's AI-assistant configuration likely references the changelog as a source of "what the project does."

**No imports** — This is a `.md` file. It is consumed by humans, CI docs generators, and possibly LLM context windows.

## Flow

There is no execution flow — this is a static document. However, the **intended reading flow** for a maintainer is:

```
1. Read the latest version block (0.18.0) → understand what's new
2. Scan "Fixed" entries across versions → understand accumulated tech debt
3. Follow (#issue, PR#) links → jump to the actual code change
4. Cross-reference with tests/ → verify the fix is covered
5. Check pyproject.toml → confirm version matches
```

The **authoring flow** (how this file gets updated) is:
```
Feature PR merged → author adds a bullet under "Added" in a new ## [X.Y.0] block
  → CI runs (tests from 0.13.0 onward) → merge → release → PyPI publish
```

## Invariants

1. **Monotonic version numbering** — Each `##` header is strictly newer than the one below it. No reordering, no duplicate versions. (Note: 0.11.0 is skipped between 0.12.0 and 0.10.0 — this is a gap, not an error, but a maintainer should know it was likely a yanked/broken release.)

2. **Date ordering** — Dates are non-increasing top-to-bottom. All entries in this file use 2026 dates, indicating a rapid development window (0.1.0 on 2026-03-23 through 0.18.0 on 2026-04-25 = ~5 weeks).

3. **Category completeness** — Every bullet must fall into exactly one of: Added, Changed, Fixed, Removed, Security. The file never mixes categories (e.g., a "Fixed" bullet doesn't sneak into "Added").

4. **CLI flag consistency** — When a flag is introduced (e.g., `--visible-to` in 0.18.0), it applies to *all* read commands listed. A maintainer adding a new read command must remember to add the flag.

5. **Tag inheritance rule** (0.18.0) — Derived nodes inherit the *union* of parent access tags. This is a semantic invariant of the data model, not just a changelog note. Code in `reasons/metadata.py` or `reasons/api.py` must enforce this.

## Error Handling

The changelog documents several error-handling decisions that a maintainer must be aware of:

- **`PermissionError`** (0.18.0) — Raised on *single-node* queries (`show`, `explain`, `trace`) when access is denied. Critically, it is **not** raised on list/search commands (those silently filter). This is an intentional asymmetry: single-node access is a "you asked for this specific thing and you can't see it" error; list-level access is "here's what you can see."
- **Nogood filtering in exports** (0.18.0) — Rather than raising an error when a nogood references a restricted node, the export *silently excludes* it. This prevents information leakage but means the exported nogood set may be incomplete.
- **`_retracted` flag idempotency** (0.16.0) — The flag is set even when the node is already OUT. This means the flag is an **audit marker**, not a state machine transition. Code that checks `node._retracted` to decide *whether* to retract will miss already-OUT nodes; code that checks it to decide *whether to log* will catch them.
- **Fixpoint iteration cap** (0.14.0) — `recompute_all` doesn't loop forever on cyclic graphs; it has a cap. A maintainer changing the propagation logic must preserve or raise this cap, or they risk infinite loops.
- **Cycle guard in depth computation** (0.15.0) — The depth calculator detects cycles. This is a precondition for the `--min-depth` / `--max-depth` filters to terminate.

---

## Topics to Explore

- [file] `reasons/api.py` — The public API layer that the changelog's CLI-centric descriptions ultimately map to. Understanding `add`, `retract`, `derive`, `propagate` as Python functions (not just CLI commands) is essential before modifying any feature listed in the changelog.
- [file] `reasons/metadata.py` — Home of the access-tags implementation (0.18.0). The tag inheritance rule ("derived nodes inherit union of parent tags") and forward propagation logic live here. Any change to the tag model must update both this file and the `--visible-to` filtering in every read command.
- [file] `reasons/repair.py` and `reasons/repair_premises.py` — The propagation/repair engine referenced by the `recompute_all` fixpoint cap (0.14.0) and the `_retracted` flag behavior (0.16.0). Understanding the fixpoint loop's termination condition is critical before touching depth filters or cycle guards.
- [file] `reasons/llm.py` and `reasons/derive.py` — The LLM-backed `derive` command (introduced 0.7.0, extended through 0.10.0 with `--exhaust`, topic filters, budgets). The changelog describes this as "propose deeper reasoning chains," but the actual prompt engineering, budget enforcement, and saturation logic are here.
- [general] `version-gap-0.11.0` — The jump from 0.10.0 (2026-03-29) to 0.12.0 (2026-04-17) skips 0.11.0. Investigating whether this was a yanked release, a CI failure, or simply an unmerged PR will reveal whether there are orphaned branches or abandoned features that should be cleaned up.

---

## Beliefs

- `access-tags-extend-belief-graph` — The 0.18.0 access-tag feature adds a second filtering dimension (RBAC) orthogonal to the existing truth-value (IN/OUT) and outlist (non-monotonic) semantics; a node can be logically IN but access-restricted, and these are independently composable.
- `recompute-all-has-iteration-cap` — The propagation engine's `recompute_all` (0.14.0) terminates via a bounded fixpoint iteration, not unbounded convergence; removing or raising the cap risks infinite loops on cyclic justification graphs.
- `retracted-flag-is-audit-not-state` — The `_retracted` flag (0.16.0) is set idempotently even when a node is already OUT, making it a monotonic audit marker rather than a state-machine transition; code must not use it as a boolean "should I retract?" gate.
- `reasons-lib-mirrors-reasons` — The `reasons_lib/` directory contains a parallel copy of every module in `reasons/`; the changelog describes features once but they must be maintained in both trees, making the dual-package structure a structural duplication risk.
- `derive-saturation-requires-exhaust-flag` — The `derive` command (0.7.0–0.10.0) performs a single LLM pass by default; only `--exhaust` (0.10.0) triggers iterative derivation until no new beliefs emerge, making the default behavior non-saturating.
