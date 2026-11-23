---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search PyPI dependents / GitHub for `import reasons_lib` before deleting, since the shim is silent.

# Explanation: Why search PyPI/GitHub for `import reasons_lib` before deleting

## The actual task

Before removing `reasons_lib/`, we need a **downstream** audit — not a repo audit. The directory in question is a 3-line compatibility shim (`reasons_lib/__init__.py`, lines 1–2):

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *
```

Nothing in the repo consumes it, so all evidence about whether it's safe to delete lives outside the repo. That evidence must be gathered from PyPI dependency graphs and GitHub code search.

## Why "the shim is silent" is the whole reason for this step

Look at what the shim does *not* do:

- It doesn't raise `DeprecationWarning`.
- It doesn't log, count, or emit any telemetry.
- It doesn't fail, print, or otherwise signal.

That means:

1. **We have no runtime signal in our own logs.** If a downstream project imports `reasons_lib`, nothing on our side (CI, release jobs, install telemetry) tells us. There is no "these users hit the shim last month" data to check before pulling the plug.
2. **Downstream users have no signal either.** They aren't warned at import time, so they will not have migrated. Deleting the shim silently breaks them on the next `pip install -U ftl-reasons`.
3. **A grep of *their* code is the only evidence.** The only way to know if the shim is load-bearing for anyone is to look at *their* code — i.e., search GitHub for `import reasons_lib` / `from reasons_lib`, and query PyPI's reverse-dependency data.

## Why the repo itself tells us nothing

Two searches confirm the shim is unused first-party:

- `find_imports_of_reasons_lib` → **0 matches** for `import reasons_lib|from reasons_lib` anywhere in the tree.
- `find_all_reasons_lib_mentions` → **17 matches**, but all of them are one of:
  - documentation (`CLAUDE.md:35` calls it "backwards-compat shim — re-exports from `reasons/`"),
  - packaging config (`pyproject.toml:31`, `packages = ["reasons", "reasons_lib"]`),
  - or an internal index (`.forge/proposed-entries.json`, lines 2–14 — just content hashes for entry naming, not code).

So the entire rationale for keeping this shim alive is **external consumers**. That's the risk the audit is buying down.

## Evidence external consumers plausibly exist

- **It ships on PyPI.** `pyproject.toml:31` includes `reasons_lib` in `[tool.setuptools] packages`. Every published `ftl-reasons` version (currently `0.58.0`, from `pyproject.toml:6`) contains the shim. Anyone who pinned any of those versions has the module importable.
- **`reasons_lib` was once a real public namespace.** `CHANGELOG.md:136` says the Python API layer was `reasons_lib.api` returning dicts — that's the shape of the *old* package, before the `reasons` rename. `CHANGELOG.md:109` shows the rename `rms` → `reasons`. So the current shim exists because at some point `reasons_lib` was the only import path.
- **The README doesn't mention it.** `read_readme_for_reasons_lib_context` returns **0 matches**. Any external user who installed via PyPI after the rename would not have been told their import path was deprecated — another reason to assume they're still using `reasons_lib` without warning.

## A subtlety that matters for the search

`from reasons import *` re-exports only names defined at the top level of `reasons/__init__.py` — `Justification`, `Node`, `Nogood`, plus anything in `__all__` if it were set. It does **not** bind submodules. So:

- `import reasons_lib; reasons_lib.Justification` — works through the shim.
- `import reasons_lib.api` or `from reasons_lib.api import ...` — **already does not work**, unless `reasons/api.py` happens to get imported elsewhere first.

This matters because your GitHub search should look for both patterns. The `reasons_lib.api` style may already be broken in the wild (not your problem), but `import reasons_lib` + attribute access may still be silently working — and will break if you delete the shim.

## What to actually run (and what's missing here)

The observations provide **no results** from PyPI or GitHub — that data has not been gathered yet. What to run:

1. **PyPI dependents (no native API — use external mirrors):**
   - `bigquery-public-data.pypi.file_downloads` filtered on `project = 'ftl-reasons'` (per-version download breakdown).
   - `libraries.io` dependents API for `ftl-reasons`.
   - `deps.dev` (Google's OSS dependency graph).
2. **GitHub code search** (`github.com/search?type=code`):
   - `"import reasons_lib"`
   - `"from reasons_lib"`
   - `"reasons_lib" filename:requirements.txt`
   - `"ftl-reasons" filename:pyproject.toml`
3. **Sourcegraph / grep.app** for mirrors not indexed by GitHub.
4. **Record the audit** (ADR / PR body / CHANGELOG entry) — otherwise the next maintainer re-asks this question.

Until at least step 1 and 2 are run, deleting `reasons_lib/` is an unmeasured breaking change.

## Topics to Explore

- [file] `pyproject.toml` — Line 31 shows `reasons_lib` is a published package, so deleting the directory actually changes the wheel contents of `ftl-reasons`, not just the source tree.
- [file] `CHANGELOG.md` — Lines 109 and 136 together document the `rms` → `reasons` rename and confirm `reasons_lib.api` was once a public API path.
- [file] `reasons_lib/__init__.py` — The silent shim itself; verify that `from reasons import *` covers top-level names but not submodules, which determines which import styles the audit must search for.
- [general] `pypi-dependents-audit` — The concrete search: BigQuery `pypi.file_downloads` for `ftl-reasons`, libraries.io dependents, and deps.dev.
- [general] `github-code-search` — The `"import reasons_lib"` / `"from reasons_lib"` queries, plus manifests (`requirements.txt`, `pyproject.toml`, `setup.cfg`) that pin `ftl-reasons`.

## Beliefs

- `reasons-lib-shim-emits-no-deprecation-signal` — `reasons_lib/__init__.py` imports from `reasons` without raising `DeprecationWarning`, logging, or emitting any other runtime signal.
- `reasons-lib-has-no-first-party-importers` — No file in the repository matches `import reasons_lib` or `from reasons_lib`, so the shim has zero internal consumers.
- `reasons-lib-is-packaged-with-ftl-reasons` — `pyproject.toml` line 31 lists `reasons_lib` in `[tool.setuptools] packages`, so the shim ships in the `ftl-reasons` wheel.
- `reasons-lib-re-exports-only-top-level-names` — `from reasons import *` re-exports names defined at the top level of `reasons/__init__.py` but does not bind submodules like `reasons_lib.api`.
- `reasons-lib-was-a-public-import-path` — `CHANGELOG.md` line 136 references `reasons_lib.api` as a caller-facing surface, confirming external code could import that namespace before the `reasons` rename.
