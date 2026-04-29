# File: CHANGELOG.md

**Date:** 2026-04-29
**Time:** 17:10



# CHANGELOG.md — Explanation

## Purpose

This is the project's **release history document**, following the [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) convention. It serves as the human-readable record of what changed between each version of `ftl-reasons` (the CLI/library). It owns the **narrative of feature evolution** — not the implementation details (that's in code and commits), but the user-facing summary of what was added, fixed, or changed in each release.

For a maintainer, this file answers: "When did feature X land?", "Which PR fixed bug Y?", and "What was in version 0.14.0?"

## Key Components

There are no classes or functions — this is a Markdown document. The "components" are structural:

- **Version headers** (`## [0.18.0] - 2026-04-25`): Each is a release, tagged with a semver number and date. There are 15 releases from 0.1.0 through 0.18.0 (note: 0.11.0 is skipped).
- **Change categories**: `### Added`, `### Fixed`, `### Changed` — the standard Keep a Changelog sections. No `### Removed` or `### Deprecated` sections appear, meaning nothing has been publicly deprecated yet.
- **Issue/PR cross-references**: Entries like `(#38, PR #42)` link changes to GitHub issues and pull requests, creating traceability from changelog to code.

## Patterns

1. **Semver with feature-driven bumps**: Minor version increments for each release. No patch releases (0.x.y) appear — the project bumps minor for everything, including pure bugfix releases (e.g., 0.16.0 is all fixes). This suggests the project is pre-1.0 and treating minor as "any notable change."

2. **Chronological descent**: Newest version at top, oldest at bottom — standard Keep a Changelog ordering.

3. **Multi-release same-day batches**: 0.15.0, 0.16.0, and 0.17.0 all share the date 2026-04-18, indicating rapid iteration with multiple logical releases cut on the same day.

4. **Issue-first development**: Most entries after 0.12.0 reference GitHub issue numbers, showing a shift toward tracked, issue-driven work.

## Dependencies

- **No code dependencies** — it's a Markdown file.
- **Consumed by**: Humans reading the repo, PyPI (if included in the package distribution via `pyproject.toml`), and the expert knowledge base (`code-expert`) which mines it for beliefs about feature timelines.
- **Maintained alongside**: `pyproject.toml` (which holds the canonical version number) and `README.md`.

## Flow

The document is append-at-top: each new release adds a section above the previous ones. The implied workflow is:

1. Accumulate changes on a branch.
2. Before tagging a release, write the changelog entry with the version and today's date.
3. Commit the changelog update as part of (or alongside) the version bump in `pyproject.toml`.

## Invariants

- Each version header must have a unique semver and date.
- Entries under `### Added` describe new user-facing capabilities; `### Fixed` describes bug corrections; `### Changed` describes breaking or notable behavior changes.
- Issue/PR references should correspond to real GitHub issues/PRs on `benthomasson/ftl-reasons`.

## Error Handling

N/A — this is a documentation file, not executable code.

## Notable Historical Details

- **0.3.0**: The project was renamed from `rms` to `reasons`, motivated by a measurable LLM accuracy improvement ("5pp LLM accuracy improvement in ablation study"). This is unusual — the rename was driven by AI-agent ergonomics, not human preference.
- **0.1.0**: The initial release already shipped a substantial feature set: full TMS with SL justifications, retraction cascades, nogood detection, dialectical argumentation, SQLite persistence, and a multi-command CLI. This was not an MVP — it was a fairly complete TMS from day one.
- **0.18.0** (latest): Adds RBAC-style access tags, which is a significant architectural addition — provenance-based filtering across the entire read path.

---

## Topics to Explore

- [file] `pyproject.toml` — Holds the canonical version number; verify it matches the latest changelog entry (0.18.0)
- [file] `reasons_lib/cli.py` — The CLI is where most changelog features surface; understanding it maps changelog entries to code
- [diff] `PR #42` — The access tags feature (0.18.0) is the largest single addition; reviewing the PR shows how a cross-cutting concern was threaded through the codebase
- [general] `version-skipping` — Version 0.11.0 is missing from the changelog; worth checking whether it was an intentional skip or an unreleased tag
- [file] `README.md` — Compare what the README promises vs. what the changelog shows was actually shipped

## Beliefs

- `changelog-follows-keep-a-changelog` — The file explicitly declares adherence to Keep a Changelog 1.1.0 format, using Added/Fixed/Changed sections with semver headers
- `no-patch-releases-exist` — All 15 releases are minor-version bumps (0.x.0); no patch-level releases (0.x.y where y > 0) have been cut
- `rename-from-rms-at-0.3.0` — The project was renamed from `rms` to `reasons` in version 0.3.0, driven by a measured 5 percentage-point LLM accuracy improvement
- `version-0.11.0-missing` — The changelog jumps from 0.10.0 to 0.12.0 with no 0.11.0 entry, indicating either a skipped version or an unrecorded release
- `access-tags-is-latest-feature` — Version 0.18.0 (2026-04-25) introduced access tags for RBAC-style provenance filtering, touching all read commands

