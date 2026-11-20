---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Determine which tree is canonical, whether `reasons_lib` is deprecated upstream or downstream, and whether a `DeprecationWarning` rollout / removal timeline exists (check `CHANGELOG.md` and `pyproject.toml`).

## Summary

### 1. Which tree is canonical: `reasons/`

`reasons/` is the canonical package. Evidence:

- `CLAUDE.md:24` — `reasons/  # importable as import reasons (also aliased as reasons_lib for backwards compat)`
- `CLAUDE.md:35` — `reasons_lib/  # backwards-compat shim — re-exports from reasons/`
- `reasons_lib/__init__.py:1-2` — the shim's own docstring: `"""Backwards-compatibility shim — use import reasons instead."""` followed by `from reasons import *`

`reasons/__init__.py` is where the real data model lives (`Justification`, `Node`, `Nogood` dataclasses, lines 6-59). `reasons_lib` contains no own types — every name resolves to the same object identity as in `reasons`.

### 2. Is `reasons_lib` deprecated upstream or downstream?

**Upstream.** `reasons_lib` is owned and shipped by this repository, not a downstream convenience alias:

- `pyproject.toml:31` — `packages = ["reasons", "reasons_lib"]` shows both trees are packaged into the single distribution `ftl-reasons` (line 6).
- The shim itself documents the intended direction ("use `import reasons` instead"), so the deprecation is declared **by the upstream maintainers** in the upstream source tree.

There is no evidence of any downstream (`pip install`-side or vendored) deprecation shim. The `.forge/proposed-entries.json:2-14` entries for `reasons_lib-api.md`, `reasons_lib-cli.md`, etc. are just generated doc-entries for the shim files — further confirming `reasons_lib` mirrors `reasons` inside this repo.

Note: the string `deprecated` also appears in `reasons/api.py:3919` (a heuristic word list for detecting "outdated/deprecated/fragile/brittle/hack/workaround" language) and in `reasons/cli.py:3064` (`--domain-id` alias) and `reasons/cli.py:3854` (`research` subcommand). **None of these relate to `reasons_lib`** — they are unrelated CLI/domain deprecations.

### 3. Is there a `DeprecationWarning` rollout / removal timeline?

**No, on all three axes:**

| Signal | Evidence | Verdict |
|---|---|---|
| Runtime `DeprecationWarning` | `reasons_lib/__init__.py:3` is just `from reasons import *  # noqa` — no `warnings.warn`, no module `__getattr__` shim | Absent |
| `CHANGELOG.md` deprecation entry | The only `reasons_lib` mention in the changelog is `CHANGELOG.md:136` in the **0.1.0** section — `Python API layer (reasons_lib.api) returning dicts...`. That's a historical origin note, not a deprecation notice. No entry in 0.3.0's "Renamed from `rms` to `reasons`" (lines 121-123) says `reasons_lib` will be removed. | Absent |
| `pyproject.toml` removal schedule | No `[tool.deprecation]`, no `[project.optional-dependencies]` split, no version pin, no `Requires-Python`-style gate — only `packages = ["reasons", "reasons_lib"]` (line 31). | Absent |

So today `reasons_lib` is a **silent** compat shim: consumers get no warning, no documented end-of-life, and no changelog promise to remove it.

### 4. What's missing / can't be answered from the observations

- Whether `reasons_lib/api.py`, `reasons_lib/cli.py`, `reasons_lib/derive.py`, `reasons_lib/storage.py`, etc. actually exist as separate files, and whether any of *those* (as opposed to `__init__.py`) emit `DeprecationWarning`. The `.forge/proposed-entries.json` entries strongly suggest they do exist, but no source was read.
- Whether any test suite asserts warning behavior (no `testpaths` entry beyond `["tests"]` in `pyproject.toml:34`).
- The **version discrepancy**: `pyproject.toml:6` says `version = "0.58.0"` while the top of `CHANGELOG.md` is `## [0.18.0] - 2026-04-25`. This means either the changelog is stale or the version was bumped out-of-band, which in turn means the "current" deprecation status is hard to pin to a release.
- No `## [Unreleased]` or `### Deprecated` section exists in `CHANGELOG.md` (only Added/Fixed/Changed), so a `DeprecationWarning` rollout would not currently have a natural home there.

## Topics to Explore

- [file] `reasons_lib/api.py` — Confirm whether the submodule re-exports via `from reasons.api import *` or wraps with a `warnings.warn(..., DeprecationWarning)`; this is the most likely place a warning would have been added if one exists.
- [file] `reasons/cli.py` — See how the project already emits or documents deprecations for real aliases (`--domain-id` at line 3064, `research` subcommand at line 3854) to infer the team's preferred deprecation mechanism.
- [file] `pyproject.toml` — Verify the packaging story: which submodules under `reasons_lib/` are actually included, and whether the `0.58.0` version is authoritative vs `CHANGELOG.md`'s `0.18.0`.
- [general] `deprecation-rollout-pattern` — Compare the `reasons_lib` shim against other deprecated aliases in the CLI to determine whether the project's convention is docstring-only deprecation or runtime warnings.
- [repo] `ftl-reasons` — Inspect release history for whether `reasons_lib` has ever been announced for removal on PyPI or in a GitHub release, since the local changelog is incomplete.

## Beliefs

- `reasons-is-canonical-tree` — `reasons/` is the canonical module tree; `reasons_lib/` is explicitly a backwards-compatibility shim that re-exports from it.
- `reasons-lib-shim-re-exports-reasons` — `reasons_lib/__init__.py:3` executes `from reasons import *`, so top-level names (`Justification`, `Node`, `Nogood`) are the same objects as in `reasons`.
- `reasons-lib-emits-no-deprecation-warning` — Neither `reasons_lib/__init__.py` nor any observed file calls `warnings.warn`/`DeprecationWarning` for the `reasons_lib` import path.
- `pyproject-ships-both-reasons-and-reasons-lib` — `pyproject.toml:31` packages `reasons` and `reasons_lib` together in the single `ftl-reasons` distribution, so the shim is maintained upstream, not by downstream consumers.
- `changelog-has-no-reasons-lib-removal-timeline` — The only `reasons_lib` mention in `CHANGELOG.md` is line 136 (0.1.0 origin note); there is no "Deprecated" or "Removed" entry with a timeline.
- `pyproject-version-disagrees-with-changelog` — `pyproject.toml:6` declares `version = "0.58.0"` while the newest `CHANGELOG.md` entry is `0.18.0`, so the changelog cannot be used as an authoritative source for current deprecation policy.
