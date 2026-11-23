---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Look for roadmap, issue labels, or semver policy that might define when compatibility shims are removed.

# Compatibility-Shim Removal Policy: Not Defined Anywhere in This Repo

**Short answer: there is no roadmap, no issue-label taxonomy, and no semver/deprecation policy that says when — or whether — the `reasons_lib` compatibility shim gets removed.** The shim exists, is still shipped, and its retirement is undocumented.

## What the observations actually show

### The shim exists and is still published

`CLAUDE.md` (Project Structure section) documents it explicitly:

```
reasons_lib/            # backwards-compat shim — re-exports from reasons/
```

and the same file notes `reasons/` is "importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)."

`pyproject.toml:31` confirms it is still a **first-class packaged artifact**, not a test fixture:

```toml
packages = ["reasons", "reasons_lib"]
```

So every `pip install ftl-reasons` still delivers the alias. Nothing in the packaging metadata hints at a sunset.

### No policy text anywhere

Two repo-wide greps came back completely empty:

| Search | Scope | Pattern | Result |
|---|---|---|---|
| `search_markdown_policy_terms` | all markdown | `roadmap\|semver\|compatibility\|shim\|deprecat\|breaking change\|removal\|remove` | **0 matches** |
| `search_python_compat_terms` | Python sources | `compat\|shim\|deprecat\|semver\|roadmap\|breaking\|remove\|legacy` | **0 matches** |

The second result is the more telling one: **not even the shim module itself carries a deprecation comment, a `DeprecationWarning`, or a "TODO: remove in X" marker.** A shim with zero in-code annotation is a shim with no intended expiry date.

Note also that `README.md` (199 lines, read in full) covers install, usage, the full command table, and test counts — but has **no** Roadmap, Versioning, Deprecation, or Contributing section. The term "compatibility" never appears in it.

### No issue-label or roadmap infrastructure in `.github`

`inspect_github_metadata` shows the entire `.github` tree is:

```
.github/workflows/tests.yml
```

That's it. Missing, relative to what a label-governed deprecation process would require:

- `.github/ISSUE_TEMPLATE/*` — no bug/feature/deprecation templates
- `.github/labels.yml` or similar — no label taxonomy to search for a `deprecation` / `breaking-change` / `good-first-removal` label
- `ROADMAP.md`, `CONTRIBUTING.md`, `GOVERNANCE.md` — none present
- `CODEOWNERS` — absent

Issue **numbers** are used heavily (`CHANGELOG.md` cites #1–#42 and PR #5–#17), so the project clearly runs on GitHub Issues — but the labels applied to them live only on the GitHub server and are **not captured by these observations**.

### No semver commitment — and the changelog claims only Keep-a-Changelog

`CHANGELOG.md` opens with:

> The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

Keep a Changelog's canonical boilerplate continues "…and this project adheres to [Semantic Versioning]." **That second half is absent here.** So the project claims a changelog *format* but explicitly does not claim a *versioning contract*. Without SemVer, there is no rule (e.g. "removals only in major bumps") to derive a shim-removal trigger from.

`pyproject.toml:11` — `requires-python = ">=3.10"` — pins the runtime floor, but nothing pins the import-path contract.

## The two real red flags

**1. Version skew between `pyproject.toml` and `CHANGELOG.md`.** The package declares `version = "0.58.0"` (`pyproject.toml:7`) while the changelog's newest heading is `## [0.18.0] - 2026-04-25`. Even if you wanted to *infer* a removal cadence from release history, the changelog is 40 minor versions stale and therefore useless as a policy source.

**2. There is no `## [Unreleased]` heading at the top of the changelog.** In the Keep-a-Changelog convention, that section is where `### Deprecated` and `### Removed` entries are staged before a release. Its absence means there is no established place for a deprecation notice to be filed — a structural hint that none has ever been planned.

**3. The shim's origin is itself undocumented.** `CHANGELOG.md` `0.3.0` records only the **`rms` → `reasons`** rename ("CLI, library, package"). No entry in the 0.1.0–0.18.0 history announces the introduction of `reasons_lib`. So the compatibility window has no recorded start date, which makes an end date even harder to justify.

Also worth discounting: `.forge/report.json` is a `diary-update` pipeline run (proposing 260 beliefs, accepting 0, `total_beliefs: 201`). It is build/automation telemetry, **not** release policy — do not read it as a roadmap artifact.

## What is missing to answer the question definitively

The observations are insufficient on exactly one point: **GitHub-side issue labels are not in the snapshot.** Labels like `deprecation`, `breaking-change`, or `v1.0` could exist server-side and govern shim removal without leaving any file in the repo. To close the gap you would need:

- `gh label list` output, or the `.github/labels.yml` that would normally sync them
- A search of open/closed issues for `reasons_lib` to see whether removal was ever discussed
- Git history of `reasons_lib/__init__.py` to determine its introduction date and whether a removal milestone was ever referenced in a commit message

Absent that, the defensible engineering conclusion is: **treat `reasons_lib` as permanent for now, because the project is pre-1.0 (`0.58.0`), claims no SemVer adherence, ships the shim in `[tool.setuptools] packages`, and has zero in-code deprecation signaling.** The conventional pre-1.0 answer — "minor bumps may break, so remove it whenever" — is available by default, but it is a convention, not a policy this repo has adopted.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm whether the shim is a bare `from reasons import *` or carries any `DeprecationWarning` / expiry comment; the grep for "compat|shim|deprecat" returned zero Python matches, which needs verifying in-scope
- [general] `github-issue-labels` — Query `gh label list` and search issues for `reasons_lib`; server-side labels are the only plausible remaining home for a shim-removal policy
- [diff] `CHANGELOG.md` vs `pyproject.toml` — Reconcile `0.18.0` (changelog head) against `0.58.0` (package version) to see which 40 releases were never documented and whether any contained a deprecation note
- [general] `reasons-lib-git-history` — `git log --follow reasons_lib/__init__.py` to establish when the alias was introduced and whether any commit references a removal milestone
- [file] `.github/workflows/tests.yml` — Check whether the shim is even exercised by CI; an untested shim is a strong practical signal it is already dead code

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — `CLAUDE.md` documents `reasons_lib/` as a backwards-compat shim re-exporting from `reasons/`, and `pyproject.toml:31` still ships it via `packages = ["reasons", "reasons_lib"]`.
- `shim-removal-has-no-documented-policy` — Repo-wide searches for `roadmap|semver|deprecat|compatibility|removal` return zero matches in both markdown and Python sources, so no removal criterion is written down.
- `changelog-lags-package-version-by-40-minors` — `pyproject.toml:7` declares `version = "0.58.0"` while `CHANGELOG.md`'s newest entry is `[0.18.0]`.
- `changelog-declares-keep-a-changelog-but-not-semver` — The changelog header cites Keep a Changelog 1.1.0 only; the usual SemVer-adherence sentence is omitted, so no versioning contract binds removals to major bumps.
- `github-directory-contains-only-ci-workflow` — `.github` holds a single entry, `.github/workflows/tests.yml`; there are no issue templates, label configs, `ROADMAP.md`, or `CONTRIBUTING.md` to encode a deprecation process.
