---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Search issues, PyPI metadata, telemetry, and downstream repos for actual `reasons_lib` imports.

## Explanation

**Bottom line:** the observations do **not** show any actual consumer import of `reasons_lib`, such as `import reasons_lib` or `from reasons_lib...`, outside the compatibility package itself. They show that `reasons_lib` is still packaged and documented as a backwards-compatibility shim, and the import direction is `reasons_lib -> reasons`, not `consumer -> reasons_lib`.

### What the local evidence shows

- `pyproject.toml` line 31 lists both packages:
  - `packages = ["reasons", "reasons_lib"]`
  - This proves `reasons_lib` is packaged. It does **not** prove anyone imports it.
- CLI entry points use the new package name:
  - `reasons = "reasons.cli:main"`
  - `reasons-mcp = "reasons.mcp_server:main"`
  - Neither entry point targets `reasons_lib.*`.
- `reasons_lib/__init__.py` lines 1-2 are an explicit shim:
  - `"""Backwards-compatibility shim — use import reasons instead."""`
  - `from reasons import *`
- The same pattern appears across `reasons_lib/api.py`, `ask.py`, `cli.py`, `derive.py`, etc. These modules import from `reasons.*`, not the other way around.
- `reasons/mcp_server.py` line 23 uses `from reasons import api`.
- `reasons/pg.py` lines 943 and 1324 use `from reasons.api import ...`.
- `CLAUDE.md` lines 24 and 35 explicitly describe `reasons_lib` as a backwards-compat alias/shim for `reasons`.
- `CHANGELOG.md` line 136 says `reasons_lib.api` existed as a Python API layer. That is historical documentation, not evidence of current external imports.
- `.git/FETCH_HEAD` line 99 and `.git/packed-refs` line 100 show a branch named `rename-reasons-lib-to-reasons`, which is evidence of migration away from the old name.
- `.forge/proposed-entries.json` lines 2-14 contain generated documentation entries named after `reasons_lib` modules. These are generated artifacts, not consumer imports.

### Search result interpretation

The `find_all_reasons_lib_references` result found 19 matches for `reasons[_-]lib`, but none are actual Python import statements outside shim/docs/config/generated files. The matches are mainly git metadata, changelog/docs, packaging metadata, and generated Forge entries.

The `find_actual_reasons_package_imports` result reports 283 matches for `from reasons|import reasons`, but that pattern is too broad to prove `reasons_lib` usage. It matches imports of `reasons`, including the shim files themselves, e.g. `reasons_lib/__init__.py:2: from reasons import *`.

### What is missing

The observations do **not** include:

- GitHub/GitLab issue search results for `reasons_lib`
- PyPI JSON metadata, download stats, or dependent-package data
- Runtime import telemetry
- Downstream repository manifests, lockfiles, or source scans

The provided telemetry files are not import telemetry:

- `costs/2026-11-19T180331-update.json` records model calls, tokens, cost, and belief counts. It has no module/import events.
- `.forge/report.json` records a `diary-update` pipeline with belief proposal/acceptance counts. It has no package-import data.

So the observations are sufficient to say **`reasons_lib` is an internal compatibility shim in this repo**, but insufficient to conclude whether any external/downstream project imports it.

## Topics to Explore

- [file] `pyproject.toml` — Confirms `reasons_lib` is still packaged in line 31 while entry points use `reasons.*`.
- [file] `reasons_lib/__init__.py` — Shows the compatibility shim implementation and re-export direction.
- [general] `pypi-ftl-reasons-download-stats` — Needed to determine whether external users install the package and which versions they use.
- [general] `github-code-search-reasons-lib` — Search downstream repos and issue bodies for real `import reasons_lib` / `from reasons_lib` statements.
- [diff] `rename-reasons-lib-to-reasons` — Inspect the branch diff referenced in `.git/FETCH_HEAD` and `.git/packed-refs` to understand the migration intent.

## Beliefs

- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/__init__.py` re-exports from `reasons` via `from reasons import *`, rather than defining an independent API.
- `pyproject-packages-includes-reasons-lib` — `pyproject.toml` line 31 lists both `reasons` and `reasons_lib` in `[tool.setuptools].packages`.
- `cli-entrypoints-use-reasons-package` — `pyproject.toml` maps `reasons` and `reasons-mcp` to `reasons.cli:main` and `reasons.mcp_server:main`, not to `reasons_lib`.
- `telemetry-has-no-import-signal` — The provided `costs/*.json` telemetry records model/token/cost and belief counts only; it contains no package-import events.
- `downstream-import-evidence-is-absent` — The observations contain no actual `import reasons_lib` or `from reasons_lib...` statement outside generated docs/config and the shim package itself.
