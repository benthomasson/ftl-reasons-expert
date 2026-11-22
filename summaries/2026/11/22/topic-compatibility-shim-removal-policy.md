---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Decide whether `reasons_lib` should remain indefinitely as a compatibility layer or eventually be removed.

## Recommendation

**Do not keep `reasons_lib` indefinitely. Keep it for now as a temporary compatibility layer, formally deprecate it, and remove it in a planned major release after a migration window.**

The evidence supports eventual removal, but **not immediate removal**.

### Why it should eventually be removed

- There are **zero internal Python references** to `reasons_lib`: `find_reasons_lib_references_in_python` returned `total_count: 0`. No Python file in the observed codebase imports the shim.
- The shim is explicitly labeled as temporary. The compatibility-note scan shows `reasons_lib/__init__.py:1`, `reasons_lib/api.py:1`, and many per-module shims all say variants of:  
  `"""Backwards-compatibility shim — use ... instead."""`
- The developer notes already describe it as a shim: `CLAUDE.md:24` says the package is “also aliased as reasons_lib for backwards compat,” and `CLAUDE.md:35` says `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/.”
- `reasons_lib/__init__.py` and `reasons_lib/api.py` are wildcard re-exports:
  - `reasons_lib/__init__.py`: `from reasons import *`
  - `reasons_lib/api.py`: `from reasons.api import *`  
  This is cheap to maintain, but it is still an extra public import surface.
- `pyproject.toml` currently ships it via:
  - `[tool.setuptools] packages = ["reasons", "reasons_lib"]`  
  So removal is not just deleting files; it also requires updating packaging.

### Why it should not be removed immediately

- `CHANGELOG.md:136` records `reasons_lib.api` historically as a Python API layer returning dicts for CLI and tool-call consumers. That suggests external consumers may have used it.
- The Python reference search only proves **internal** non-use; it does not prove no downstream/package consumers exist.
- `README.md` does not mention `reasons_lib` anywhere in the observed lines `1-198`, so the current primary docs do not promise it. But absence from README is not proof that external code never imported it.
- The shim files are tiny, so keeping them during a deprecation window is low cost compared with an unannounced break.

### Recommended removal plan

1. Add a runtime deprecation warning in `reasons_lib/__init__.py`, e.g. `warnings.warn("reasons_lib is deprecated; use reasons", DeprecationWarning, stacklevel=2)`, so `import reasons_lib` and `import reasons_lib.api` surface the message.
2. Document the migration in `README.md` and `CHANGELOG.md`: use `import reasons` and `from reasons.api import ...` instead of `reasons_lib`.
3. Keep the shim for at least one clearly documented deprecation cycle.
4. Remove it in a major-version release, update `pyproject.toml` to drop `reasons_lib` from `[tool.setuptools] packages`, and update `CLAUDE.md` references.
5. Before removal, audit external usage: PyPI consumers, GitHub imports of `reasons_lib`, downstream issues, and any internal release/testing infrastructure not captured by the Python search.

### What is missing

The observations do not include downstream package usage, GitHub-wide import data, deprecation-warning tests, or maintainer compatibility policy. Those are needed to choose the exact removal date, not to decide whether indefinite retention is necessary.

## Topics to Explore

- [file] `pyproject.toml` — Shows that `reasons_lib` is packaged; removal requires changing `[tool.setuptools] packages` as well as deleting shim files.
- [file] `CHANGELOG.md` — Line 136 records historical `reasons_lib.api` usage, which informs how long the deprecation window should be.
- [file] `README.md` — Current public docs do not mention `reasons_lib`; a migration note should be added before removal.
- [general] `external-usage-audit` — Search downstream repositories and package consumers for `import reasons_lib` and `from reasons_lib.api` to quantify breakage risk.
- [general] `deprecation-warning-implementation` — Determine how to emit and test `DeprecationWarning` from a wildcard-reexport shim package.

## Beliefs

- `reasons-lib-is-not-used-internally` — The gathered Python search for `reasons_lib` returned zero matches across the codebase.
- `reasons-lib-shims-reexport-reasons-modules` — `reasons_lib/__init__.py` re-exports `reasons`, and `reasons_lib/api.py` re-exports `reasons.api`, using wildcard imports.
- `reasons-lib-is-packaged` — `pyproject.toml` includes `"reasons_lib"` in `[tool.setuptools] packages`.
- `changelog-documents-reasons-lib-api-as-python-api-layer` — `CHANGELOG.md:136` records `reasons_lib.api` as a Python API layer returning dicts for CLI and tool-call consumers.
- `readme-omits-reasons-lib` — The observed `README.md` content does not mention `reasons_lib`.
