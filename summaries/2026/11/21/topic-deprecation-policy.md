---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Decide whether to keep the shim silently, add warnings, or schedule removal.

# Deciding on the `reasons_lib` Shim

## What the code actually contains

`reasons_lib/` is a full parallel package: every module is a 2-line stub that re-exports the real implementation. For example `reasons_lib/api.py`:

```python
"""Backwards-compatibility shim — use ``from reasons.api`` instead."""
from reasons.api import *  # noqa: F401,F403
```

The `search_shim_deprecation_warnings` observation shows the same pattern repeated across ~30+ modules (`api.py`, `ask.py`, `cluster.py`, `compact.py`, etc.), each with a self-describing docstring but **no runtime warning**. `pyproject.toml:32` ships it alongside the real package:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

## The decisive evidence

Three observations narrow the choice:

1. **Zero internal usage.** `find_imports_of_reasons_lib` → `"total_count": 0`. Nothing inside the repo imports the shim. The two `from reasons …` imports in `reasons/mcp_server.py:23` and `reasons/pg.py:943,1324` target the *real* package. So removal will not break this codebase.
2. **`reasons` is the new name, not the old one.** `CHANGELOG.md:136` describes `reasons_lib.api` as the original Python API layer, and `CLAUDE.md:24` says the code is "importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)". The shim is therefore a rename artefact — external consumers who wrote `import reasons_lib` before the rename are the only possible dependents.
3. **Docs already point users away from it.** Nothing in `README.md` mentions `reasons_lib`; only dev-facing docs (`CLAUDE.md:24,35`) and the historical CHANGELOG entry do. So no new user is being onboarded to the shim.

What we **cannot** see from these observations:

- PyPI download counts per module (imports aren't tracked).
- GitHub code search for `import reasons_lib` outside this repo.
- The project's semver policy for pre-1.0 releases (version is `0.58.0`).

## Recommendation: **add warnings now, schedule removal, do not keep silently**

Silent keeping is the wrong default here. The shim costs almost nothing to ship (a handful of two-line files), but every release without a warning resets the clock on consumers who think they have time. Removing silently is worse — `ImportError` at upgrade time is a hard break with no migration window.

A staged plan:

### 1. Add a `DeprecationWarning` in `reasons_lib/__init__.py`

```python
import warnings
warnings.warn(
    "reasons_lib is deprecated; use `import reasons` instead. "
    "See CHANGELOG.md for removal timeline.",
    DeprecationWarning,
    stacklevel=2,
)
```

Because Python always imports a parent package before a submodule, `import reasons_lib.api` will also trigger this warning. `DeprecationWarning` is filtered out by default outside `__main__`, so ordinary users are not spammed — but test suites running with `-W error::DeprecationWarning` will catch it, which is exactly the audience you want to reach. Do **not** use `print()` or `logging.warning()` here: those break UIs and scripts.

### 2. Write the removal schedule into `CHANGELOG.md`

Pick a concrete target. Given the pre-1.0 version and zero internal usage, a reasonable line is: *"Remove `reasons_lib` in the next minor release (0.60.0)."* That gives external users one full minor cycle to migrate. Bump it in the docstring in each `reasons_lib/*.py` shim as part of the same change (currently they say "Backwards-compatibility shim — use …" but do not name a version).

### 3. Survey external usage before the deadline

- PyPI stats won't show per-module imports, but `pip download ftl-reasons` and grep in downstream packages will.
- GitHub code search for `"from reasons_lib"` / `"import reasons_lib"`.
- If usage is non-trivial, extend the deadline; if it's zero, remove in 0.60.0.

### 4. On removal

Delete the whole `reasons_lib/` tree and update `pyproject.toml:32` to `packages = ["reasons"]`. The CHANGELOG should get a "Removed" entry under the Unreleased header.

## Why this beats the alternatives

| Option | Cost | Risk |
| --- | --- | --- |
| **Keep silently** | ~0 | Consumers never migrate; shim lives forever |
| **Remove silently** | ~0 | Hard `ImportError` for external users; no CHANGELOG trail |
| **Warn + schedule** | ~5 lines + a CHANGELOG paragraph | Rare `DeprecationWarning` noise under `-W error`; delayed cleanup |

The shim is small, harmless, and self-documenting — the only real risk is drift between `reasons_lib/*.py` and `reasons/*.py` as the real package evolves. Adding a `from reasons import *` to a shim that silently picks up new symbols is a maintenance distractor. Scheduling removal retires that burden after one cycle.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The single file where a `DeprecationWarning` should be added; because Python imports parent packages first, this one warning covers every `reasons_lib.*` submodule.
- [file] `CHANGELOG.md` — Where the removal schedule and "Removed" entry must be recorded; line 136 contains the historical entry that oriented `reasons_lib.api` as the old name.
- [file] `pyproject.toml` — Line 32 explicitly lists `reasons_lib` in `[tool.setuptools].packages`; removal requires editing this and re-uploading.
- [general] `deprecation-policy` — The project is at `0.58.0` with no README mention of `reasons_lib`; worth deciding whether pre-1.0 minor bumps are allowed to remove shims or must wait for 1.0.
- [general] `external-usage-survey` — PyPI stats and GitHub code search for `from reasons_lib` / `import reasons_lib` would replace the "unknown external consumers" caveat with concrete data before committing to 0.60.0.

## Beliefs

- `reasons-lib-shim-is-not-imported-internally` — `find_imports_of_reasons_lib` reports `total_count: 0`, so no file inside this repo depends on `reasons_lib`.
- `reasons-lib-ships-with-package` — `pyproject.toml:32` includes `reasons_lib` in `[tool.setuptools].packages`, so every wheel contains the shim.
- `reasons-lib-re-exports-reasons` — Every shim module is a one-line `from reasons.X import *` (e.g., `reasons_lib/api.py:2`, `reasons_lib/cli.py:2`), making `reasons` the single source of truth.
- `reasons-replaces-reasons-lib` — `CHANGELOG.md:136` names `reasons_lib.api` as the original API layer, and `CLAUDE.md:24` describes `reasons` as the current name, so `reasons` is a rename, not the other way around.
- `reasons-lib-emits-no-warning` — None of the `reasons_lib/*.py` shims contain a `warnings.warn` or `logging.warning` call; the "deprecation" lives only in the module docstring.
