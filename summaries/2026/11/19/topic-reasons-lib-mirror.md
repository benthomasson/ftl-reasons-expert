---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Why both `reasons/` and `reasons_lib/` contain this file; likely a packaging shim but worth confirming how they stay in sync.

# Why `reasons/` and `reasons_lib/` both exist

## Short version

`reasons_lib/` is a **backward-compatibility shim package**, not a duplicate implementation. Its files contain a docstring and a single `from reasons.<module> import *` line. They stay "in sync" by construction: they don't hold any logic of their own, they just re-export whatever the canonical `reasons/` package currently exposes.

## Evidence

**1. The docstrings say it verbatim.**

- `reasons_lib/__init__.py` (3 lines):
  ```python
  """Backwards-compatibility shim — use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```
- `reasons_lib/api.py` (3 lines):
  ```python
  """Backwards-compatibility shim — use ``from reasons.api`` instead."""
  from reasons.api import *  # noqa: F401,F403
  ```

Compare to the canonical versions:
- `reasons/__init__.py` (59 lines) actually defines `Justification`, `Node`, and `Nogood` dataclasses.
- `reasons/api.py` is 5,886 lines and defines `init_db`, `add_node`, `retract_node`, etc.

There is no duplicated logic in `reasons_lib/`. The whole package is a re-export facade.

**2. `pyproject.toml` ships both, but only `reasons` is wired up as the entry point.**

- Line `[tool.setuptools] packages = ["reasons", "reasons_lib"]` — both directories are included in the wheel so that both import paths work post-install.
- Entry points target the canonical package only:
  ```toml
  [project.scripts]
  reasons     = "reasons.cli:main"
  reasons-mcp = "reasons.mcp_server:main"
  ```
  There is no `reasons_lib.cli:main` entry point.
- The distribution is named `ftl-reasons` while the import package is `reasons` — a strong hint that the project was historically distributed under a different import name (likely `reasons_lib`) and `reasons` became canonical later.

**3. Nothing inside the repo imports `reasons_lib`.**

The `find_reasons_lib_references` search returned `total_count: 0` across the codebase. That means the shim exists solely for **external consumers** still writing `import reasons_lib` or `from reasons_lib.api import …`. Internally, everything (CLI, API, dataclasses) uses `reasons.*`.

**4. The `api.py` shim depends on `reasons/__init__.py` already being fully loaded.**

In `reasons/api.py` line 21 there is `from . import Justification`. Because `reasons_lib/api.py` first triggers `from reasons.api import *`, `reasons/__init__.py` runs to completion (defining `Justification`) before `api.py` executes — so the import order is fine.

## How they actually stay in sync

They don't require manual maintenance. Three rules govern the contract:

1. **Name-level sync is automatic.** Because `reasons_lib/__init__.py` uses `from reasons import *`, any new public name added to `reasons/__init__.py` becomes available as `reasons_lib.<name>` immediately. Same for `reasons_lib/api.py → reasons.api`.
2. **Submodule-level sync is not automatic.** `import reasons_lib.api` works only because a physical `reasons_lib/api.py` file exists. If `reasons/` gained a new submodule (say `reasons/network.py`), the old namespace `reasons_lib.network` would only resolve if someone also created `reasons_lib/network.py` with the same 3-line shim. The `import *` in `__init__.py` does **not** publish submodules.
3. **`__all__` becomes the contract if introduced.** Neither `reasons/__init__.py` (all 59 lines) nor the first 200 lines of `reasons/api.py` define `__all__`. Without an `__all__`, `from reasons.api import *` re-exports every non-underscore name in the module's namespace — including incidental top-level imports such as `copy`, `json`, `os`, `sqlite3`, `sys`, `re`, `logging`, `Path`, `datetime`, `timezone`, `combinations`, `Callable`, plus `Justification`, `pubsub`, `build_meta`, `Network`, `Storage`, and `PROPOSALS_SCHEMA`. If a future refactor ever adds an explicit `__all__` to `reasons.api`, the shim's surface silently narrows to that list.

## What we cannot confirm from these observations

- **Which shims actually exist inside `reasons_lib/`.** Only `__init__.py` and `api.py` were read. We don't have a directory listing, so we can't tell whether other public modules (`cli`, `storage`, `network`, `pg`, `mcp_server`, `metadata`, `pubsub`) also have shims. If they don't, then `import reasons_lib.storage` may fail for external consumers even though `import reasons.storage` works. This is the biggest gap in the "how do they stay in sync" question.
- **Whether `reasons/api.py` defines `__all__` somewhere after line 200** (of 5,886). If it does, that is the effective compatibility contract for `reasons_lib.api`.
- **Whether any test asserts equality between `dir(reasons)` and `dir(reasons_lib)`.** Nothing in the gathered observations verifies this. The CI workflow at `.github/workflows/tests.yml` was listed but not read, so we can't claim there's no check.
- **The commit that introduced the rename.** No git history was collected; the `ftl-reasons` distribution name is the only clue that a rename happened.

## Topics to Explore

- [file] `reasons_lib/` — List the directory to confirm which `reasons/*` submodules actually have shim files and which do not (only `__init__.py` and `api.py` were observed).
- [file] `reasons/api.py` — Search the full 5,886-line file for `__all__`; it determines the effective re-export contract for `reasons_lib.api`.
- [file] `.github/workflows/tests.yml` — Check whether CI runs anything that exercises `reasons_lib` (e.g., an import smoke test) or only `reasons`.
- [general] `package-rename-history` — `git log --follow reasons_lib/__init__.py` and inspect `pyproject.toml` history to confirm when `reasons` became canonical and whether `reasons_lib` is a rename or an alias-for-distribution-name.
- [file] `tests/` — Look for any test that imports `reasons_lib` or asserts that both namespaces expose the same symbols; without one, the shim is essentially unverified.

## Beliefs

- `reasons-lib-is-a-compat-shim` — Both `reasons_lib/__init__.py` and `reasons_lib/api.py` describe themselves as "Backwards-compatibility shim" in their docstrings and contain only a `from reasons … import *` statement.
- `reasons-lib-reexports-reasons-via-star-import` — The shim files contain no duplicated logic; they forward all public names via `from reasons import *` and `from reasons.api import *`, so name-level additions to `reasons` are automatically visible through `reasons_lib`.
- `reasons-lib-submodule-parity-requires-physical-shim-files` — A `reasons_lib.<submodule>` import path resolves only if a physical `reasons_lib/<submodule>.py` shim exists; the `import *` in `__init__.py` does not expose submodules.
- `pyproject-declares-both-packages-for-wheel-inclusion` — The only non-docstring reference to `reasons_lib` is `packages = ["reasons", "reasons_lib"]` in `pyproject.toml`, ensuring both import paths ship in the wheel.
- `reasons-is-the-canonical-entrypoint-package` — Console scripts `reasons` and `reasons-mcp` both target `reasons.*` (`reasons.cli:main`, `reasons.mcp_server:main`); no entry point targets `reasons_lib`.
