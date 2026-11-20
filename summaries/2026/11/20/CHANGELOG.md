---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# File: CHANGELOG.md

# `CHANGELOG.md`

> TL;DR — This is a **release ledger**, not executable code. It is a hand-maintained Markdown document that records every user-visible change to the `reasons` project between `0.1.0` and `0.18.0`, following the [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) convention. It has no imports, no consumers in the build graph, and no runtime behavior — but it is the authoritative history of the CLI/API surface.

---

## 1. Purpose

`CHANGELOG.md` owns exactly one responsibility: **document what changed between releases, in human-readable form, newest-first**.

It answers, for a maintainer or user:
- *"When was `--visible-to` added?"* → `0.18.0`, 2026-04-25
- *"Which release introduced `derive`?"* → `0.7.0`
- *"Why is the package called `reasons`?"* → `0.3.0` (`Changed`: renamed from `rms` after a 5pp LLM-accuracy ablation)

It is **not** an API reference (that's `README.md` / `CLAUDE.md`), **not** a task tracker (that's the GitHub issue numbers embedded in bullets), and **not** a SemVer authority (that lives in `pyproject.toml`). It is a chronological superset of *"things a user would notice."*

---

## 2. Key Components

Because this is Markdown, the "components" are structural elements, not symbols:

| Element | Form | Contract |
|---|---|---|
| File header | `# Changelog` + preamble (lines 1–5) | Declares the format contract: Keep a Changelog 1.1.0. Every entry below must obey it. |
| Release heading | `## [X.Y.Z] - YYYY-MM-DD` | One per release, reverse-chronological. Square-bracket prefix enables GitHub auto-linking when a matching git tag exists. |
| Change category | `### Added` / `### Changed` / `### Fixed` | Only these three of Keep a Changelog's six categories are used (`Deprecated`, `Removed`, `Security` are absent). |
| Change bullet | `- <description> (#issue, PR #N)` | One user-visible change. The trailing `(#N)` / `(#N, PR #M)` is an issue/PR reference, not a Markdown link (bare `#N`). |
| Inline version tag | `... (0.8.0)` / `... (0.9.0)` inside `0.10.0` | Escape hatch for entries that were never given their own heading. |

The vocabulary inside the bullets mirrors real CLI identifiers in `reasons/cli.py`:

- Commands: `add`, `add-justification`, `retract`, `assert`, `status`, `show`, `explain`, `trace`, `search`, `list`, `lookup`, `nogood`, `propagate`, `log`, `derive`, `accept`, `what-if`, `supersede`, `deduplicate`, `import-beliefs`, `import-agent`, `import-json`, `sync-agent`, `export`, `export-markdown`, `export-card`, `compact`, `check-stale`, `hash-sources`, `convert-to-premise`, `challenge`, `defend`, `trace-access-tags`
- Flags: `--visible-to`, `--access-tags`, `--any`, `--exhaust`, `--min-depth`, `--max-depth`, `--premises`, `--has-dependents`, `--reason`, `--version`
- Domain terms: SL justification, outlist, nogood, entailment/dependency chain, entrenchment, TMS, fixpoint

That vocabulary is what makes this file useful — grep here to find *"when did this flag exist?"*

---

## 3. Patterns & Conventions

1. **Keep a Changelog + SemVer-in-spirit.** Versions are `0.x.y`; no `1.0.0` yet. Date is ISO-8601 `YYYY-MM-DD` and matches the release date, not the commit date.
2. **Reverse chronology.** New sections are prepended above the previous top entry (`0.18.0` is line 7). This is the standard pattern — it means a diff to this file almost always shows an `@@ -5,+N @@` hunk at the top.
3. **Category-before-version ordering.** Within a version, `Added` precedes `Fixed` precedes `Changed`. No entry mixes categories out of order.
4. **Issue-first traceability.** Bullets that close a GitHub issue carry `(#N)` or `(#N, PR #M)`. That pattern indicates this project links source-control work to release notes.
5. **Fold-in escape hatch.** `0.10.0` retroactively absorbs `0.8.0` and `0.9.0`, marking their bullets inline. This is used when versions are cut but not documented as they happen — a mild violation of the format's "one heading per version" rule.
6. **Grouped thematic entries in dense releases.** `0.18.0`'s `Added` block is 10 bullets, effectively a mini spec for the RBAC feature (`access_tags`, inheritance, propagation, `PermissionError`, nogood filtering in exports). This is where the changelog edges toward "feature narrative."

---

## 4. Dependencies

- **Imports:** none — the file is Markdown.
- **Imported by:** none in source. Nothing in `reasons/`, `reasons_lib/`, or `tests/` reads it.
- **Implicit dependents:**
  - `pyproject.toml` — the canonical version string; the changelog must agree with it per release.
  - `README.md` / `CLAUDE.md` — likely reference the "current" version or the rename history.
  - PyPI release page (`ftl-reasons`) — the project uses PyPI release notes; this file is the internal source.
  - `.github/workflows/*` — if a release workflow parses the top heading, it would break on malformed `## [x.y.z] - date` headings.
- **Referenced by prose, not code:** issue/PR numbers in bullets point back to GitHub — this is a *reverse* dependency on the issue tracker existing.

---

## 5. Flow

There is no control flow — but the *reading flow* is deliberate:

```
top of file
  │
  ├── Preamble (format contract)
  │
  ├── 0.18.0 (2026-04-25)  ← newest
  ├── 0.17.0 (2026-04-18)
  ├── 0.16.0 (2026-04-18)
  ├── 0.15.0 (2026-04-18)
  ├── 0.14.0 (2026-04-17)
  ├── 0.13.0 (2026-04-17)
  ├── 0.12.0 (2026-04-17)
  ├── 0.10.0 (2026-03-29)  ← note: 0.11.0 absent
  ├── 0.7.0  (2026-03-28)  ← note: 0.8.0 / 0.9.0 absent
  ├── 0.6.0  (2026-03-24)
  ├── 0.5.0  (2026-03-24)
  ├── 0.4.0  (2026-03-23)
  ├── 0.3.0  (2026-03-23)  ← rename; only Changed
  ├── 0.2.0  (2026-03-23)  ← Added + Fixed
  └── 0.1.0  (2026-03-23)  ← initial release, longest Added
bottom of file
```

Notice the temporal clustering: `0.1.0`–`0.7.0` all land in a 7-day window (2026-03-23 → 2026-03-28), then a six-day gap, then `0.10.0` (2026-03-29), then a ~19-day gap, then `0.12.0`–`0.18.0` clustered over 9 days (2026-04-17 → 2026-04-25). This is a fast-iteration project; the changelog absorbs high release frequency by grouping.

Practical reading recipe:
1. Skim `## [0.18.0]` — this is the current feature frontier (RBAC `access_tags`).
2. Jump to `## [0.3.0]` — the rename `rms` → `reasons` explains why the library path is `reasons_lib` and the PyPI package is `ftl-reasons`, two non-obvious names.
3. Scan `## [0.1.0]` — this is effectively a description of the TMS core (nodes, SL justifications, outlist, nogoods, backtracking, dialectical `challenge`/`defend`).

---

## 6. Invariants

Rules the file is *supposed* to uphold (and where it currently slips):

1. **Newest entry at the top.** Prepend-only edits. ✅ upheld.
2. **`## [X.Y.Z] - YYYY-MM-DD` heading shape.** Required for tooling/tag-linking. ✅ upheld for every present entry.
3. **Version number matches `pyproject.toml`.** Unverifiable from this file alone, but implied by the format.
4. **One heading per released version.** ❌ **Violated:** no `[0.11.0]` heading exists; `0.8.0` and `0.9.0` appear only as inline `(0.8.0)` / `(0.9.0)` annotations inside `0.10.0` bullets.
5. **Categories are a subset of Keep a Changelog's six.** Only `Added` / `Changed` / `Fixed` used; `Deprecated`, `Removed`, `Security` never appear — even in a release like `0.16.0` that moves a value from antecedent to outlist (arguably a `Changed`) and filters metadata keys (arguably `Removed`).
6. **Every `Added` bullet names a user-facing capability** (command, flag, or documented behavior) — not an internal refactor. ✅
7. **Every `Fixed` bullet names the symptom**, with a `(#N, PR #M)` reference where one exists. ✅ (see `0.14.0`, `0.13.0`).
8. **Prior versions are immutable.** Entries should not be edited after publish — except for the deliberate `0.8.0`/`0.9.0` fold-in, which is the one sanctioned mutation.

---

## 7. Error Handling

There is no error handling in the code sense — no exceptions, no return codes. The relevant failure mode is **documentation drift**, and it manifests in three concrete places:

| Drift | Where | Consequence |
|---|---|---|
| Missing version heading | `0.11.0` between `0.10.0` (line 55) and `0.12.0` (line 45) | A release exists in history with no user-facing record. Anyone bisecting `git log` for `0.11.0` gets nothing. |
| Folded versions | `0.8.0` / `0.9.0` inside `0.10.0` `Added` bullets | Breaks any script that assumes `## [x.y.z]` headings are one-per-release. Parsers should tolerate inline version tags or they will under-count releases. |
| Category under-reporting | `0.16.0` has only `Fixed` but contains a semantic change (outlist supersession) and a removal (internal metadata keys filtered from JSON export) | A reader filtering for `Changed`/`Removed` will miss these. |

Mitigation patterns a maintainer should apply:
- Add new entries **at the top only**, and add a heading even for trivial patches.
- When publishing, ensure `## [pyproject.version]` exists — a release script can assert this.
- Use `Changed` for behavior changes that aren't bug fixes (`0.3.0` rename correctly uses `Changed`; `0.16.0` arguably should have too).
- Keep `(#N, PR #M)` references so the changelog can be reconciled against closed issues.

---

## Topics to Explore

- [file] `pyproject.toml` — Confirm the current version (should match `0.18.0` or newer) and the `ftl-reasons` package name established in `0.3.0`.
- [file] `reasons/cli.py` — The aggregate of every command named in this changelog; use it to verify that bullets like `add-justification`, `sync-agent`, `trace-access-tags` actually exist with the documented flags.
- [general] `access-tags-rbac` — `0.18.0` documents inheritance, forward propagation, `PermissionError` on single-node reads, and nogood filtering in exports; trace how `--visible-to` is threaded through `show`/`list`/`search`/`export` in `reasons/cli.py` and `reasons_lib/storage.py`.
- [diff] `0.3.0-rename` — The `rms` → `reasons` rename (with the 5pp ablation rationale) explains the dual `reasons/` and `reasons_lib/` directories; inspect the tree at that tag to see what moved.
- [general] `release-checklist` — Whether a script asserts `## [<version>]` exists at the top of `CHANGELOG.md` before publishing to PyPI; this is what would have caught the missing `0.11.0` entry.

---

## Beliefs

- `changelog-follows-keep-a-changelog-1-1-0` — The preamble (lines 1–5) explicitly cites Keep a Changelog 1.1.0, and only its `Added`, `Changed`, and `Fixed` categories are used.
- `changelog-is-reverse-chronological` — Entries run from `[0.18.0] - 2026-04-25` at the top down to `[0.1.0] - 2026-03-23` at the bottom.
- `version-0-11-0-has-no-changelog-heading` — No `## [0.11.0]` section exists between `[0.10.0]` and `[0.12.0]`.
- `versions-0-8-0-and-0-9-0-are-folded-into-0-10-0` — Their features appear as bullets annotated `(0.8.0)` and `(0.9.0)` inside the `[0.10.0]` `Added` section rather than as standalone headings.
- `access-tags-rbac-is-introduced-in-0-18-0` — The `access_tags` field, `--visible-to`, tag inheritance, forward propagation, and `PermissionError` on single-node reads are all first documented under `[0.18.0]`.
- `project-renamed-from-rms-to-reasons-in-0-3-0` — `[0.3.0]` records the rename across CLI, library, and package, and the PyPI publication as `ftl-reasons`.
- `changelog-uses-issue-and-pr-references` — Fixed/feature bullets carry `(#N)` or `(#N, PR #M)` markers (e.g., `(#16, PR #17)`, `(#11, PR #14)`) linking entries to GitHub work items.
