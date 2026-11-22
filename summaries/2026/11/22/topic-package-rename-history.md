---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: `git log --follow reasons_lib/__init__.py` and inspect `pyproject.toml` history to confirm when `reasons` became canonical and whether `reasons_lib` is a rename or an alias-for-distribution-name.

## What the current observations establish

The current files strongly indicate that **`reasons` is the canonical import package** and **`reasons_lib` is a backwards-compatibility shim**, but they do **not** tell us the exact commit/date when that happened.

Evidence:

- `pyproject.toml` line 6 declares the distribution name as `ftl-reasons`.
- `README.md` line 20 says `pip install ftl-reasons`, confirming the distribution name is not `reasons` or `reasons_lib`.
- `pyproject.toml` line 31 packages both import paths:
  ```toml
  packages = ["reasons", "reasons_lib"]
  ```
  So `reasons_lib` is a packaged top-level Python package shipped by the `ftl-reasons` distribution.
- The console scripts point at the canonical package:
  - `pyproject.toml` line 27: `reasons = "reasons.cli:main"`
  - `pyproject.toml` line 28: `reasons-mcp = "reasons.mcp_server:main"`
- `reasons_lib/__init__.py` lines 1-2 explicitly say:
  ```python
  """Backwards-compatibility shim — use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```
- `CLAUDE.md` line 24 says `reasons/` is importable as `import reasons` and “also aliased as reasons_lib for backwards compat.”
- `CLAUDE.md` line 35 says `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/.”
- `reasons/__init__.py` contains the actual data model, including `Justification` around line 7, `Node` around line 27, and `Nogood` around line 53.

So, from the current state:

- **Canonical import package:** `reasons`
- **Backwards-compatible import alias/shim:** `reasons_lib`
- **Distribution name:** `ftl-reasons`
- **`reasons_lib` is not a distribution-name alias.** It is an import-package shim.

## What `git log --follow reasons_lib/__init__.py` would confirm

`git log --follow` is the right tool to distinguish a **rename** from an **alias/shim introduced later**.

You would inspect for:

- An `A`/added commit showing `reasons_lib/__init__.py` was created as a shim.
- An `R`/rename commit showing it was moved from another path.
- The commit message and diff showing whether `reasons` was introduced as the replacement implementation or whether `reasons_lib` was replaced by a shim.
- Whether `reasons_lib` was ever a full implementation rather than just a wildcard re-export.

Useful commands:

```bash
git log --follow --name-status -- reasons_lib/__init__.py
git log --follow -p -- reasons_lib/__init__.py
```

That would answer “was `reasons_lib` renamed, or was it left behind as an alias?”

## What `pyproject.toml` history would confirm

The observations include only the current `pyproject.toml`, not its history. To confirm **when** `reasons` became canonical, inspect changes to:

- `[project].name` at `pyproject.toml` line 6
- `[project.scripts]` at `pyproject.toml` lines 27-28
- `[tool.setuptools].packages` at `pyproject.toml` line 31

Useful commands:

```bash
git log -p -- pyproject.toml
git log -S 'packages = ["reasons", "reasons_lib"]' -- pyproject.toml
git log -S 'reasons = "reasons.cli:main"' -- pyproject.toml
git log -S 'name = "ftl-reasons"' -- pyproject.toml
```

What to look for:

- If older history shows `packages = ["reasons_lib"]` and scripts like `reasons_lib.cli:main`, then a later commit switching to `reasons` is the canonicalization point.
- If `pyproject.toml` always shipped both packages, then `reasons_lib` may have been introduced as a compatibility shim from the start of that packaging layout.
- If `[project].name` previously differed from `ftl-reasons`, that is a separate distribution-name change, not the same thing as the import-package alias.

## Important gap

The observations do not include:

- actual `git log --follow` output,
- any historical version of `pyproject.toml`,
- commit hashes/dates/version tags for the canonicalization change.

Therefore we can confirm the **current architecture**:

> `reasons` is canonical; `reasons_lib` is a backwards-compatibility shim shipped by the `ftl-reasons` distribution.

But we cannot confirm the exact **when** or whether there was an earlier rename without the missing git history.

Also note: `CHANGELOG.md` line 136 still references `reasons_lib.api`. Since `reasons_lib/__init__.py` only does `from reasons import *`, that does not automatically prove `import reasons_lib.api` still works. You should inspect whether `reasons_lib/api.py`, `reasons_lib/cli.py`, etc. also exist as shims.

## Topics to Explore

- [general] `git-log-follow-history` — Run `git log --follow --name-status -- reasons_lib/__init__.py` to see whether the shim was added directly or renamed from a fuller implementation.
- [file] `pyproject.toml` — Trace `[project].name`, `[project.scripts]`, and `[tool.setuptools].packages` through `git log -p -- pyproject.toml` to identify the canonicalization commit and version.
- [file] `CHANGELOG.md` — Read around line 136 to see the version/date context for the `reasons_lib.api` reference and whether it predates the shim.
- [general] `reasons-lib-submodules` — Check whether legacy subpaths such as `reasons_lib.api`, `reasons_lib.cli`, and `reasons_lib.mcp_server` exist as shims or only the top-level `__init__.py` does.
- [file] `reasons/__init__.py` — Confirm exactly which names `from reasons import *` re-exports from the canonical package.

## Beliefs

- `ftl-reasons-distribution-name` — `pyproject.toml` line 6 declares the distribution name as `ftl-reasons`; `README.md` line 20 confirms installation via `pip install ftl-reasons`.
- `reasons-is-canonical-import-package` — `reasons_lib/__init__.py` lines 1-2 and `CLAUDE.md` lines 24/35 identify `reasons` as canonical and `reasons_lib` as a backwards-compatible re-export.
- `reasons-lib-is-not-a-distribution-name` — `reasons_lib` appears only as a packaged top-level package in `pyproject.toml` line 31, not as `[project].name`.
- `ftl-reasons-packages-both-import-paths` — `pyproject.toml` line 31 includes both `reasons` and `reasons_lib`, so the distribution ships both import paths.
- `reasons-lib-reexports-top-level-reasons-names` — `reasons_lib/__init__.py` line 2 uses `from reasons import *`, so top-level names from `reasons/__init__.py`, such as `Justification`, `Node`, and `Nogood`, are available through `reasons_lib` unless `reasons` defines `__all__`.
