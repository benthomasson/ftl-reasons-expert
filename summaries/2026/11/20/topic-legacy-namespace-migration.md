---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Whether `reasons_lib` is formally deprecated, when it can be removed, and whether a warning should be emitted.

## Short answer

`reasons_lib` is **not formally deprecated** in the observed code. It is a deliberately packaged backwards-compatibility shim, and its docstring recommends `import reasons`, but there is no `DeprecationWarning`, no changelog deprecation entry, and no deprecation marker specific to `reasons_lib`.

It **should not be removed yet**: `pyproject.toml:31` still ships it as an installed package (`packages = ["reasons", "reasons_lib"]`), and `CLAUDE.md:35` describes it as the “backwards-compat shim.” Safe removal requires a formal deprecation cycle first.

A warning should be emitted **only if the project intends to remove it**. If removal is planned, add a `DeprecationWarning` in `reasons_lib/__init__.py`, document it in `CHANGELOG.md`, and keep it for at least one release/major-version window per the project’s compatibility policy.

## Evidence

- `reasons_lib/__init__.py` contains only a docstring:  
  `"""Backwards-compatibility shim — use ``import reasons`` instead."""`  
  followed by `from reasons import *`.
- `CLAUDE.md:35` calls `reasons_lib/` a “backwards-compat shim — re-exports from `reasons/`.”
- `CLAUDE.md:24` says `reasons/` is “also aliased as `reasons_lib` for backwards compat.”
- `pyproject.toml:31` still includes `"reasons_lib"` in `[tool.setuptools].packages`, so it is part of the published package.
- `CHANGELOG.md:136` only mentions `reasons_lib.api` historically in the 0.1.0 entry; it is not a deprecation notice.
- The deprecation-marker search found only unrelated matches:
  - `reasons/api.py:3919` — vocabulary list containing `'deprecated'`
  - `reasons/cli.py:3064` — deprecated alias for `--domain-id`
  - `reasons/cli.py:3854` — deprecated `research` CLI alias  
  None are about `reasons_lib`.

So the shim is **legacy/backwards-compatible**, but not **formally deprecated**.

## When can it be removed?

Not now. A safe removal sequence would be:

1. Declare deprecation explicitly in docs and `CHANGELOG.md`.
2. Add a warning at import time in `reasons_lib/__init__.py`.
3. Publish at least one release with that warning.
4. Check downstream usage if possible.
5. Remove `"reasons_lib"` from `pyproject.toml:31`, delete the package, and add a removal changelog entry.

The exact removal version cannot be determined from the observations because the project’s compatibility policy is not shown. If the project follows semver, removal should likely happen in a major release after a deprecation cycle. If it treats pre-1.0 versions as unstable, a minor release may be allowed, but that policy is not visible here.

## Should a warning be emitted?

Currently: **no warning is emitted**, and that is consistent with the shim not being formally deprecated.

If removal is intended, yes: emit a `DeprecationWarning` from the shim, for example:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
import warnings

warnings.warn(
    "reasons_lib is deprecated; use `import reasons` instead",
    DeprecationWarning,
    stacklevel=2,
)

from reasons import *  # noqa: F401,F403
```

Note that `DeprecationWarning` is hidden by default outside `__main__` and tests, so the warning should be paired with changelog and documentation updates.

## What is missing

The observations do not establish:

- The project’s versioning/compatibility policy.
- Whether `reasons_lib` has submodules such as `reasons_lib.api` that also need shims or warnings.
- Whether external users import `reasons_lib`; repo-local references alone are not proof of no downstream usage.
- Whether there is an existing deprecation policy or release-note convention to follow.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The shim itself; this is where a `DeprecationWarning` and migration message would live.
- [file] `pyproject.toml` — Declares `reasons_lib` as a packaged import path; removal requires changing `packages`.
- [file] `CHANGELOG.md` — Needs an explicit “Deprecated” entry before any removal, and later a “Removed” entry.
- [general] `deprecation-policy` — Determine the expected semver/support window and warning cycle for public import paths.
- [general] `downstream-reasons-lib-usage` — Check issues, telemetry, or external repos before deleting a published import path.

## Beliefs

- `reasons-lib-is-packaged` — `pyproject.toml:31` includes `"reasons_lib"` in `[tool.setuptools].packages`, so it ships in the installed distribution.
- `reasons-lib-reexports-reasons` — `reasons_lib/__init__.py` uses `from reasons import *`, making it a compatibility facade over `reasons`.
- `reasons-lib-has-no-deprecation-warning` — `reasons_lib/__init__.py` does not emit a `DeprecationWarning`; it only contains a docstring telling users to import `reasons`.
- `deprecation-marker-search-finds-no-reasons-lib-deprecation` — The observed `deprecat` matches are unrelated CLI/API strings at `reasons/api.py:3919`, `reasons/cli.py:3064`, and `reasons/cli.py:3854`.
- `reasons-lib-removal-requires-packaging-update` — Because `reasons_lib` is listed in `pyproject.toml`, deletion also requires removing it from packaging metadata.
