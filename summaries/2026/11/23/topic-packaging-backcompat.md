---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Verify that `reasons_lib` is included in wheels/sdists and whether external callers rely on it.

## Verification: `reasons_lib` in wheels/sdists and external caller reliance

### 1. Is `reasons_lib` shipped in wheels and sdists? — Yes

The packaging is driven by `pyproject.toml`:

- **Build backend** — `setuptools.build_meta` (lines 1–3).
- **Package list** — `[tool.setuptools]` on line 31 declares:
  ```toml
  packages = ["reasons", "reasons_lib"]
  ```
  This is the authoritative list for both sdist and wheel assembly. There is no `find_packages`/`find:` auto-discovery here, so the two listed packages are the *complete* set that will be shipped. Both `reasons_lib/__init__.py` and `reasons_lib/` (whatever else is inside it — see gaps below) are therefore guaranteed to land in the built wheel and sdist.

- `reasons_lib` is not declared in `[project.optional-dependencies]`, `[project.scripts]`, or elsewhere — so it is always included regardless of extras.

**One caveat that the observations do not fully resolve:** whether *submodules* of `reasons_lib` (e.g., `reasons_lib/api.py`) exist and are shipped. `setuptools.packages` will include any `.py` files underneath a listed package directory, so if `reasons_lib/api.py`, `reasons_lib/cli.py`, etc. exist on disk, they ship too. If only `__init__.py` exists, that is the only file shipped. The observations only show `reasons_lib/__init__.py`.

### 2. What does `reasons_lib` actually contain? — Currently a thin shim

`reasons_lib/__init__.py` is 3 lines and does nothing but re-export:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

`CLAUDE.md` codifies this:
- Line 24: `reasons/  # importable as import reasons (also aliased as reasons_lib for backwards compat)`
- Line 35: `reasons_lib/  # backwards-compat shim — re-exports from reasons/`

So the shipping intent is clear: `reasons_lib` exists purely for backward compatibility with the older import path `import reasons_lib`.

### 3. Do external callers rely on it? — Inconclusive from these observations

The evidence splits three ways:

**Evidence that something has historically referenced `reasons_lib` submodules**
- `CHANGELOG.md` line 136: *"Python API layer (`reasons_lib.api`) returning dicts for CLI and tool-call consumers."* The phrase "tool-call consumers" implies downstream consumers outside the package.
- `.forge/proposed-entries.json` (lines 2–14) enumerates 13 doc entries named after `reasons_lib` submodules — `reasons_lib-api`, `reasons_lib-cli`, `reasons_lib-check_stale-check_stale`, `reasons_lib-network-_justification_valid`, `reasons_lib-storage`, etc. These mirror the `reasons/` submodules and strongly suggest a moment in time when `reasons_lib/` was populated with real submodule files (or was the canonical documented import path).

**Evidence against active external reliance**
- The entire `README.md` (199 lines) never mentions `reasons_lib`. The install/usage sections all say `import reasons` only.
- Only one changelog entry references `reasons_lib` historically, versus a current `__init__.py` that is a 3-line shim.
- `CLAUDE.md` labels it explicitly as a shim.

**A consistency problem worth flagging**
`from reasons import *` in `reasons_lib/__init__.py` only re-exports names defined or imported at `reasons/__init__.py`'s top level (the three dataclasses: `Justification`, `Node`, `Nogood`). It does **not** create attributes like `reasons_lib.api`, `reasons_lib.cli`, or `reasons_lib.storage` — those would only resolve if `reasons_lib/api.py`, `reasons_lib/cli.py`, etc. exist as *separate files* under the `reasons_lib/` directory. Because the observations only show `reasons_lib/__init__.py`, the CHANGELOG and `.forge` references cannot be confirmed against the current tree. Either:
1. Those submodules exist on disk (not captured), and external callers truly can do `from reasons_lib.api import ...`; or
2. Those references are stale documentation, and the shim is now inert beyond `import reasons_lib`.

### 4. What's missing to fully answer "do external callers rely on it?"

- **Full listing of `reasons_lib/`** (is it just `__init__.py`, or does it still have `api.py`, `cli.py`, `network.py`, `storage.py`, `check_stale.py`, `compact.py`, `derive.py`, `import_agent.py`?).
- **Any test file** under `tests/` that does `import reasons_lib` — this would show whether the shim's public surface is exercised internally; without one, the shim has no regression coverage.
- **Downstream evidence** — search PyPI/GitHub for `from reasons_lib import` or `import reasons_lib`. Not present in the gathered observations.
- **Git history** — when `reasons_lib/api.py` stopped being the canonical API surface, and whether a deprecation timeline/`DeprecationWarning` was ever in place (none is visible in the current `__init__.py`).

## Topics to Explore

- [file] `reasons_lib/` — Full directory listing will confirm whether `api.py`, `cli.py`, `network.py`, `storage.py`, etc. still exist as shims, or whether the `.forge` entries are stale docs.
- [file] `reasons/__init__.py` — `from reasons import *` semantics depend on what names are exported here (currently only `Justification`, `Node`, `Nogood`); check for `__all__`.
- [general] `reasons-lib-deprecation-timeline` — Whether a `DeprecationWarning` or removal date is planned; `CHANGELOG.md:136` is the only historical pointer.
- [general] `downstream-consumers-of-reasons-lib` — External search (PyPI dependents, GitHub code search) for `from reasons_lib` to quantify real breakage risk if the shim were removed.
- [file] `CHANGELOG.md` — Line 136 and surrounding entries establish when `reasons_lib.api` was the documented API layer versus now.

## Beliefs

- `packaging-includes-reasons-lib` — `[tool.setuptools] packages = ["reasons", "reasons_lib"]` in `pyproject.toml:31` guarantees both packages are present in every wheel and sdist produced by `setuptools.build_meta`.
- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` contains only a docstring plus `from reasons import *`, providing no independent logic.
- `reasons-lib-api-is-not-derivable-from-init` — Because `reasons/__init__.py` exposes only `Justification`, `Node`, and `Nogood`, `reasons_lib.api` (as referenced in `CHANGELOG.md:136`) can only resolve if a separate `reasons_lib/api.py` exists — the `from reasons import *` star-import alone is insufficient.
- `readme-does-not-document-reasons-lib` — The 199-line `README.md` documents only `import reasons` and never mentions `reasons_lib`.
- `reasons-lib-deprecation-has-no-runtime-warning` — No `DeprecationWarning` (or any code) appears in `reasons_lib/__init__.py`, so the shim is silent at import time.
