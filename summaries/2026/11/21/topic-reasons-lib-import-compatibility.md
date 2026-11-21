---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Search external docs, tests, and release notes for `reasons_lib.api` and `reasons_lib` usage.

# Search Results: `reasons_lib.api` and `reasons_lib` Usage

## What the code actually contains

The `reasons_lib` package is a **pure backwards-compatibility shim**. Both files are 3 lines and contain no logic of their own:

- `reasons_lib/__init__.py` — `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py` — `from reasons.api import *  # noqa: F401,F403`

Both docstrings explicitly say "use `import reasons` / `from reasons.api` instead," confirming this package exists only for name preservation, not active use.

## Where it's referenced

Three references exist, all in docs, none in executable code:

1. **`CHANGELOG.md:136`** (v0.1.0 entry): *"Python API layer (`reasons_lib.api`) returning dicts for CLI and tool-call consumers"* — this is the historical origin. `reasons_lib.api` was the original name of the Python API layer.
2. **`CLAUDE.md:24`** — `reasons/` is noted as "importable as `import reasons` (also aliased as reasons_lib for backwards compat)".
3. **`CLAUDE.md:35`** — `reasons_lib/` is described as a "backwards-compat shim — re-exports from `reasons/`."

The CHANGELOG at **v0.3.0** records: *"Renamed from `rms` to `reasons` (CLI, library, package)"* — this is the rename that made `reasons_lib` obsolete. Before 0.3.0 the import path was `reasons_lib`, and `reasons` became canonical afterwards.

## What tests and imports show

- `find_reasons_lib_python_usages` returned **0 matches** — no code in the repo currently imports `reasons_lib` (as a module, class, string, or attribute).
- `tests/test_api.py:7` uses `from reasons import api`, i.e. tests exercise the canonical path, **not** the shim. So the shim has **no dedicated test coverage** in the observed files.
- `pyproject.toml:32` explicitly includes the shim in the wheel: `packages = ["reasons", "reasons_lib"]` — it is still shipped to PyPI, so external consumers relying on `reasons_lib.api` still work.

## What the release notes do and do not say

- `CHANGELOG.md` shows the *introduction* at 0.1.0 and the *rename* at 0.3.0, but there is **no entry recording the addition of the `reasons_lib` shim itself** (it was created implicitly when 0.3.0 renamed the canonical package).
- The shipped `pyproject.toml` version is `0.58.0`, but the CHANGELOG stops at `0.18.0` — there is a **large gap in release hygiene**; anything after 0.18.0 is undocumented in this file.
- The `README.md` never mentions `reasons_lib` — so public-facing docs already treat `reasons` as the only import path.

## Gaps / what's missing

- **No test asserts `reasons_lib.api is reasons.api`** (or that `reasons_lib` re-exports attributes). A stale or broken shim would go undetected.
- **No `DeprecationWarning`** is emitted by either shim file; the intent to migrate is only in docstrings, so downstream users get no runtime nudge.
- **No alternative docs source** (e.g. ReadTheDocs, generated API docs, GitHub wiki) is present in the repo to search — the "external docs" for `reasons_lib.api` effectively consist of the CHANGELOG entry and CLAUDE.md notes only.
- **Full `CHANGELOG.md` history beyond 0.18.0 was not in the observation set**, so any later removal/deprecation decision about `reasons_lib` is not visible here.

## Topics to Explore

- [file] `tests/test_api.py` — The only test file for the functional API; check whether it contains an assertion that `reasons.api` is importable as `reasons_lib.api` (none in the first 200 lines shown).
- [file] `reasons_lib/__init__.py` — Verify the shim is a simple re-export; consider adding a `DeprecationWarning` here if you want to nudge users toward `reasons`.
- [general] `changelog-0-18-to-0-58` — Recover or regenerate the missing CHANGELOG entries from 0.18.0 through 0.58.0 to confirm whether `reasons_lib` was ever formally deprecated.
- [function] `reasons/api.py:__all__` — Confirm what `from reasons.api import *` actually re-exports so the shim's surface area is well-defined.
- [repo] `PyPI-wheel-contents` — Inspect a published `ftl-reasons==0.58.0` wheel to verify `reasons_lib/` is still included (should be, per `pyproject.toml:32`).

## Beliefs

- `reasons-lib-is-backwards-compat-shim` — `reasons_lib` re-exports `reasons` via `from reasons import *` and contains no independent logic.
- `reasons-lib-api-originated-in-v0-1-0` — `reasons_lib.api` was the original Python API import path (CHANGELOG:136), made redundant by the v0.3.0 rename to `reasons`.
- `reasons-lib-ships-with-package` — `pyproject.toml` includes `"reasons_lib"` in `[tool.setuptools].packages`, so it is installed alongside `reasons`.
- `reasons-lib-has-no-test-coverage` — No observed test imports `reasons_lib`; `tests/test_api.py:7` uses `from reasons import api` instead.
- `reasons-lib-emits-no-deprecation-warning` — Neither shim file raises `DeprecationWarning`, so consumers are only informed via docstrings.
- `changelog-is-stale-after-0-18` — `pyproject.toml` version is `0.58.0` while `CHANGELOG.md` ends at `0.18.0`, leaving 40 minor releases undocumented.
