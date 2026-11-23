---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Check git history / issue tracker for a planned `reasons_lib` removal (version, date, or deprecation-warning addition) to confirm the end state is removal rather than permanent dual maintenance.

## Finding

I cannot confirm a planned `reasons_lib` removal from the observations provided. The bundle contains **no git history, tags, branches, commits, issues, PRs, or roadmap data**, so the requested “version/date/deprecation-warning” check was not actually possible from the evidence.

What the codebase observations do show is that `reasons_lib` is currently documented as a **backwards-compatibility shim**, not as a deprecated module with a removal target.

## Evidence from the observations

- `CHANGELOG.md:136` mentions `reasons_lib` only historically:
  - `- Python API layer (`reasons_lib.api`) returning dicts for CLI and tool-call consumers`
  - This is under `[0.1.0]`, not a later removal/deprecation entry.
- `CLAUDE.md:24` says:
  - `reasons/ # importable as `import reasons` (also aliased as reasons_lib for backwards compat)`
- `CLAUDE.md:35` says:
  - `reasons_lib/ # backwards-compat shim — re-exports from reasons/`
- The deprecation search found **no `reasons_lib`-related deprecation warning**. The three matches are unrelated:
  - `reasons/api.py:3919` — a word list containing `"deprecated"`, not a warning.
  - `reasons/cli.py:3064` — deprecated alias for `--domain-id`.
  - `reasons/cli.py:3854` — deprecated `research` command alias.
- The Python search for `reasons_lib` returned zero matches in the searched scope.
- `.forge/proposed-entries.json` contains generated/proposed entries dated `2026/04/23` for `reasons_lib-*`, but this is not an issue tracker or roadmap and does not indicate removal.

## Interpretation

From these files alone, the inferred end state is **not “removal planned”**. It is closer to:

> `reasons_lib` is a maintained backwards-compatibility shim that re-exports from `reasons/`.

That is also not “permanent dual maintenance” in the sense of duplicated logic: `CLAUDE.md:35` says the shim re-exports from `reasons/`. So the current evidence supports a compatibility alias, not two independently maintained APIs.

## What is missing

To confirm whether the end state is removal rather than a permanent shim, we need evidence not present here:

- `git log --all -- reasons_lib`
- `git log --all -S'reasons_lib' --oneline`
- `git log --all --grep='reasons_lib'`
- Git tags and release notes after `0.18.0`
- GitHub/GitLab issue or PR search for `reasons_lib`, `backwards compat`, and `removal`
- Direct inspection of `reasons_lib/__init__.py` and any `warnings.warn(..., DeprecationWarning)` calls
- Packaging metadata such as `pyproject.toml` / `setup.py` to see whether `reasons_lib` is still shipped

## Topics to Explore

- [file] `CHANGELOG.md` — Release history through `0.18.0`; confirms no later `reasons_lib` removal or deprecation entry appears.
- [file] `CLAUDE.md` — Lines 24 and 35 are the strongest current statement that `reasons_lib` is a backwards-compat shim.
- [general] `git-history-reasons-lib` — Run `git log -S'reasons_lib' --all` and `git log --all -- reasons_lib` to find actual add/deprecate/remove commits.
- [general] `issue-tracker-reasons-lib` — Search issues/PRs for `reasons_lib` and “removal” to find roadmap intent not captured in docs.
- [file] `reasons/api.py` — Check line 3919 context; it is a word list, not a `reasons_lib` deprecation warning.

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — `CLAUDE.md:35` documents `reasons_lib/` as a backwards-compat shim that re-exports from `reasons/`.
- `reasons-lib-is-aliased-to-reasons` — `CLAUDE.md:24` says `reasons` is also aliased as `reasons_lib` for backwards compatibility.
- `reasons-lib-has-no-deprecation-warning` — The deprecation search found no `reasons_lib`-related `DeprecationWarning`; the three matches are unrelated.
- `changelog-contains-no-reasons-lib-removal-entry` — `CHANGELOG.md` mentions `reasons_lib` only at line 136 under `0.1.0` and has no later removal/deprecation entry through `0.18.0`.
- `reasons-lib-python-search-returned-zero-matches` — The Python search for `reasons_lib` returned zero matches in the searched scope.
