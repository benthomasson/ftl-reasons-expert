---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How the `reasons_lib` → `reasons` migration is being sequenced; whether the shims are scheduled for removal and what deprecation warnings exist.

# `reasons_lib` → `reasons` migration sequencing

The migration appears to follow an **implementation-first, compatibility-shim** pattern:

1. The canonical implementation lives in `reasons/` and `reasons.api`.
2. `reasons_lib/` remains as a thin backwards-compatibility facade:
   - `reasons_lib/__init__.py:1-2` — docstring says “Backwards-compatibility shim — use `import reasons` instead.” and then does `from reasons import *`.
   - `reasons_lib/api.py:1-2` — docstring says “use `from reasons.api` instead.” and then does `from reasons.api import *`.
3. Packaging still ships both packages:
   - `pyproject.toml` includes `packages = ["reasons", "reasons_lib"]` under `[tool.setuptools]`.
4. The observed internal reference search for `reasons_lib` returned **0 matches**, suggesting no in-repo code still imports the shim. The remaining mentions are the shim files themselves and docs:
   - `CLAUDE.md:24` — notes that `reasons/` is importable as `import reasons` and “also aliased as reasons_lib for backwards compat.”
   - `CLAUDE.md:35` — explicitly calls `reasons_lib/` a “backwards-compat shim — re-exports from reasons/”.
   - `CHANGELOG.md:136` — older changelog entry references `reasons_lib.api` from before/around the earlier API layer.

So the sequence visible in the code is: **new package implemented → old namespace kept as re-export shim → packaging continues to include both → no observed removal phase yet.**

## Are the shims scheduled for removal?

**No evidence of removal scheduling is present in the observations.**

There is no target version, date, issue reference, or release-note entry indicating that `reasons_lib` will be removed. `pyproject.toml` still packages `reasons_lib`, and the docs only describe it as a backwards-compatibility shim rather than a deprecation-with-deadline.

Caveat: `pyproject.toml` says version `0.58.0`, while the top of `CHANGELOG.md` in the observation is `0.18.0`, so the changelog may not reflect the latest migration/removal planning. The observations are sufficient to say no removal is scheduled **in the shown files**, but insufficient to prove there is no external roadmap issue.

## What deprecation warnings exist?

For the `reasons_lib` import shims: **none at runtime**. Neither shim file calls `warnings.warn`, raises `DeprecationWarning`, or emits any import-time warning. The only migration guidance is the soft deprecation in the docstrings.

The only observed `deprecated` matches are unrelated to the package shims:

- `reasons/api.py:3919` — appears to be part of a word/phrase list containing `'outdated', 'deprecated', 'fragile', ...`, not a warning call.
- `reasons/cli.py:3064` — help text: `"(deprecated alias for --domain-id)"`.
- `reasons/cli.py:3854` — CLI subparser help: `Alias for 'repair' (deprecated)`.

So there are **CLI-level deprecation notices**, but **no runtime deprecation warnings for `reasons_lib`**.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm there are no import-time warnings or side effects beyond the star re-export.
- [file] `pyproject.toml` — Check how long both `reasons` and `reasons_lib` remain in the setuptools package list.
- [file] `CHANGELOG.md` — Look for any later release entry that defines a shim removal version or migration deadline.
- [general] `reasons-lib-external-usage` — Search downstream users, tests, and docs for remaining `reasons_lib` imports before planning removal.
- [function] `reasons/cli.py:main` — Inspect CLI alias/deprecation handling separately from the package-name migration.

## Beliefs

- `reasons-lib-is-compatibility-shim` — `reasons_lib/__init__.py` and `reasons_lib/api.py` are thin star-import shims over `reasons` and `reasons.api`.
- `packaging-ships-both-reasons-and-reasons-lib` — `pyproject.toml` lists both `reasons` and `reasons_lib` in `[tool.setuptools] packages`.
- `reasons-lib-imports-emit-no-deprecation-warning` — Neither observed `reasons_lib` shim file calls `warnings.warn` or emits a `DeprecationWarning`.
- `cli-deprecations-are-unrelated-to-reasons-lib` — The deprecated CLI aliases at `reasons/cli.py:3064` and `reasons/cli.py:3854` concern `--domain-id` and `research`/`repair`, not package imports.
- `reasons-lib-removal-not-scheduled-in-observed-docs` — The observed docs describe `reasons_lib` as backwards-compatible but contain no removal date or target version.
