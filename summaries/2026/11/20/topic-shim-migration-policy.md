---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The project-level policy for retiring `reasons_lib.*` mirrors: are they deprecated with warnings, silently supported, or slated for deletion?

## Policy

**Short answer: silently supported as a backwards-compatibility shim.** The observed project policy is not “deprecated with warnings,” and there is no evidence that `reasons_lib.*` is slated for deletion.

### Evidence

1. **The shim itself emits no warning.**
   - `reasons_lib/__init__.py:1` is a bare re-export:
     ```python
     from reasons import *  # noqa: F401,F403
     ```
   - There is no `warnings.warn(...)`, `DeprecationWarning`, or `FutureWarning` in the observed shim.

2. **Deprecation grep finds nothing for `reasons_lib`.**
   - The only Python “deprecat” matches are unrelated:
     - `reasons/api.py:3919` — a text token in a list
     - `reasons/cli.py:3064` — deprecated CLI flag alias
     - `reasons/cli.py:3854` — deprecated CLI command alias
   - None of these are `reasons_lib` import deprecations.

3. **Documentation calls it a compatibility shim, not a removal target.**
   - `CLAUDE.md:24` says `reasons/` is “also aliased as reasons_lib for backwards compat.”
   - `CLAUDE.md:35` says `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/.”
   - `CHANGELOG.md:136` references the historical `reasons_lib.api` API layer.
   - No README or CHANGELOG entry announces a `reasons_lib` removal, sunset, or warning policy.

4. **Packaging still includes the shim.**
   - `pyproject.toml`’s `[tool.setuptools]` section includes:
     ```toml
     packages = ["reasons", "reasons_lib"]
     ```
   - That is consistent with active support rather than deletion.

### Caveat

This is inferred from the observed files. There is no explicit written retirement policy in the provided observations. Also, the observed `reasons_lib/__init__.py` only shows top-level mirroring via `from reasons import *`; whether specific submodules like `reasons_lib.api` are individually mirrored is not visible from the provided evidence.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the full shim surface and whether any submodule-specific mirroring exists beyond the observed `from reasons import *`.
- [file] `pyproject.toml` — Inspect packaging configuration; removing `reasons_lib` would require changing the `[tool.setuptools] packages` list.
- [general] `backwards-compatibility-policy` — Look for roadmap, issue labels, or semver policy that might define when compatibility shims are removed.
- [file] `CHANGELOG.md` — Check whether future releases mention `reasons_lib` deprecation or removal.
- [general] `reasons-lib-import-tests` — Search tests for imports of `reasons_lib` to see whether compatibility behavior is explicitly tested.

## Beliefs

- `reasons-lib-shim-is-bare-reexport` — `reasons_lib/__init__.py:1` contains only `from reasons import *` and emits no warning.
- `reasons-lib-is-packaged` — `pyproject.toml` includes both `reasons` and `reasons_lib` in `[tool.setuptools] packages`.
- `reasons-lib-has-no-observed-deprecation-warning` — Grep for “deprecat” in Python files found no match in `reasons_lib`.
- `reasons-lib-is-documented-as-backwards-compat-shim` — `CLAUDE.md:24` and `CLAUDE.md:35` describe it as an alias/backwards-compat shim.
- `reasons-lib-deletion-is-not-announced` — Observed README, CHANGELOG, and retirement-language searches contain no removal or sunset notice for `reasons_lib`.
