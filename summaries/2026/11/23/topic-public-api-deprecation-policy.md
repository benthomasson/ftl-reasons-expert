---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Determine whether `reasons_lib` is an externally supported path that requires warnings before removal.

## Determination

**Yes — treat `reasons_lib` as an externally supported backwards-compatibility import path. Removal should be preceded by deprecation warnings.**

The code evidence shows that `reasons_lib` is not just an internal alias. It is packaged, documented as a compatibility shim, and referenced in public-facing docs/changelog. The absence of internal Python usages does **not** prove there are no external consumers, because `ftl-reasons` is a published package and `reasons_lib` is included in the distribution.

### Evidence that `reasons_lib` is externally supported

- `pyproject.toml` includes both packages in `[tool.setuptools]`:
  - `packages = ["reasons", "reasons_lib"]`
  - This means `reasons_lib` is installed with the distribution, so external users can import it.
- `reasons_lib/__init__.py:1-2` explicitly says:
  - `"""Backwards-compatibility shim — use ``import reasons`` instead."""`
  - `from reasons import *`
  - This is a compatibility layer, not an internal-only implementation detail.
- Documentation confirms the compatibility intent:
  - `CLAUDE.md:24`: `reasons/` is importable as `import reasons` and "also aliased as reasons_lib for backwards compat"
  - `CLAUDE.md:35`: `reasons_lib/` is a "backwards-compat shim — re-exports from reasons/"
- `CHANGELOG.md:136` references `reasons_lib.api`, which suggests `reasons_lib` has had a documented public surface beyond just the package root.

### Evidence about warnings and removal risk

- A search for deprecation warning patterns found no warnings in `reasons_lib`:
  - `deprecation_warning_patterns.total_count = 3`
  - The matches are in `reasons/api.py:3919` and `reasons/cli.py:3064,3854`, not in `reasons_lib/__init__.py`.
  - So the shim currently emits no `DeprecationWarning` or `FutureWarning`.
- A search for `reasons_lib` Python usages found zero matches:
  - `reasons_lib_python_usages.total_count = 0`
  - This means there is no internal Python code importing `reasons_lib`, but this does not establish that external users do not import it.
- `.forge/proposed-entries.json` lists entries such as:
  - `entries/2026/04/23/reasons_lib-api.md`
  - `entries/2026/04/23/reasons_lib-network.md`
  - `entries/2026/04/23/reasons_lib-storage.md`
  - These are proposed documentation entries, not source files, but they reinforce that `reasons_lib` was treated as a meaningful public path.

### Conclusion

`reasons_lib` is **externally visible and explicitly maintained for backwards compatibility**. Therefore, removing it would be a breaking change for any external consumer using `import reasons_lib` or subpaths such as `reasons_lib.api`.

If removal is planned, the project should first add a deprecation warning to `reasons_lib`, document the removal timeline, and allow a deprecation window. The current code does **not** do that.

Caveat: the observations do not include a formal deprecation policy, PyPI consumer data, or a full `reasons_lib/` directory listing. So we can determine that `reasons_lib` is externally supported in practice, but we cannot prove from these observations alone whether project policy formally requires a warning period.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the shim behavior and verify whether adding an import-time `DeprecationWarning` would affect `from reasons_lib import ...` users.
- [file] `pyproject.toml` — Check packaging lines around `[tool.setuptools]` to confirm `reasons_lib` is shipped in built wheels and sdists.
- [general] `reasons_lib-public-surface` — Determine whether submodules like `reasons_lib.api` and `reasons_lib.network` currently exist or are only historical/proposed entries.
- [repo] `deprecation-policy` — Look for project-level rules on compatibility shims, warning periods, and removal timelines.
- [file] `CHANGELOG.md` — Trace releases mentioning `reasons_lib` to understand whether it was publicly announced and how long it has been supported.

## Beliefs

- `reasons-lib-is-packaged` — `pyproject.toml` includes `reasons_lib` in `[tool.setuptools] packages`, so it is part of the installed distribution.
- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/__init__.py` documents itself as a backwards-compatibility shim and re-exports via `from reasons import *`.
- `reasons-lib-import-emits-no-warning` — `reasons_lib/__init__.py` contains no `warnings.warn`, `DeprecationWarning`, or `FutureWarning`.
- `reasons-lib-has-documented-public-usage` — `CHANGELOG.md:136` references `reasons_lib.api`, and `CLAUDE.md:24,35` describes `reasons_lib` as a backwards-compatible alias.
- `reasons-lib-has-no-internal-python-usages` — A codebase search for `reasons_lib` found zero Python import usages, which does not rule out external consumers.
