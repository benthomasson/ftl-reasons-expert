---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search external packaging/dependency metadata off-repo to gauge how many consumers still rely on `reasons_lib`, which determines how much longer the shim must ship.

### What this task is asking

“Search external packaging/dependency metadata off-repo” means looking outside this repository for evidence of real downstream usage: PyPI release/download data, reverse-dependency indexes, public code search, issue reports, or telemetry. The goal is to estimate how many external consumers still do `import reasons_lib` or `from reasons_lib...`, because that determines when the compatibility shim in `reasons_lib/__init__.py` can be removed.

### What the repo already tells us

- The shim is currently shipped in the installable package:
  - `pyproject.toml` line 31 includes both `"reasons"` and `"reasons_lib"` in `[tool.setuptools] packages`.
  - `pyproject.toml` line 8 shows the current version is `0.58.0`.
- The shim itself is minimal:
  - `reasons_lib/__init__.py` lines 1-3 contain only a backwards-compatibility docstring and `from reasons import *`.
- The real implementation lives in `reasons/__init__.py`, which defines `Justification`, `Node`, and `Nogood` (lines 1-59).
- There are no internal Python imports of `reasons_lib`:
  - `grep_reasons_lib_in_python` returned `total_count: 0`.
- Documentation describes it as legacy compatibility:
  - `CLAUDE.md` line 24 says `reasons/` is also aliased as `reasons_lib` for backwards compatibility.
  - `CLAUDE.md` line 35 says `reasons_lib/` is a backwards-compat shim.
- Historical context suggests `reasons_lib` may have had real consumers:
  - `CHANGELOG.md` line 136 mentions an old Python API layer at `reasons_lib.api`.
  - The `[0.3.0]` changelog entry says the project was renamed from `rms` to `reasons`, so off-repo searches should also consider older distribution/import names, not only `ftl-reasons` and `reasons_lib`.
- The README does not mention `reasons_lib`; it only documents installing `ftl-reasons` and using the CLI. So current public docs do not encourage new users to import the shim.

### What the observations do not provide

The observations are entirely local repo data. They do **not** include:

- PyPI download statistics for `ftl-reasons`.
- Reverse-dependency data from Libraries.io, deps.dev, GitHub dependency graph, or similar.
- Public code-search results for `import reasons_lib`, `from reasons_lib`, `reasons_lib.api`, etc.
- Evidence from GitHub issues, support requests, deprecation warnings, or opt-in telemetry.
- Distribution/version breakdown showing whether consumers are on old releases or current releases.

Therefore, the observations are insufficient to answer “how many consumers still rely on `reasons_lib`.” The local repo proves the shim exists and is still packaged, but it does not quantify external usage.

### What an actual off-repo search should collect

1. **PyPI package metadata and download stats**
   - Search for `ftl-reasons` on PyPI.
   - Use `pypistats.org` or PyPI JSON metadata as a rough popularity signal.
   - Note that downloads are not import-specific and include CI/caches.

2. **Reverse-dependency indexes**
   - Check Libraries.io, deps.dev, GitHub dependency graph, or similar for projects depending on `ftl-reasons`.
   - These indexes usually track distribution names, not Python import names, so they will not directly show `reasons_lib` imports.

3. **Public code search**
   - Search GitHub/GitLab/Grep.app for:
     - `import reasons_lib`
     - `from reasons_lib`
     - `reasons_lib.api`
     - `from reasons_lib import`
   - Exclude this repository itself.
   - Also search older names such as `rms` if the `[0.3.0]` rename is relevant.

4. **Issue/support evidence**
   - Search GitHub issues, discussions, Stack Overflow, and package metadata for mentions of `reasons_lib`.
   - If a deprecation warning is added to `reasons_lib/__init__.py`, monitor reports for a release cycle.

### Practical decision framing

If off-repo search finds no public imports, no reverse dependencies, and no support evidence, the shim can likely be deprecated with a warning and removed in a future breaking release. If any external consumers are found, the shim should remain until they migrate or until an explicit deprecation window has elapsed. Without off-repo evidence, removing it from `pyproject.toml` line 31 would be a guess rather than a measured migration decision.

## Topics to Explore

- [general] `PyPI reverse-dependency search for ftl-reasons` — Find distributions that declare a dependency on `ftl-reasons`; this measures package-level dependents, not direct `reasons_lib` imports.
- [general] `GitHub code search for import reasons_lib` — Estimate public downstream usage of the shim by searching for `import reasons_lib`, `from reasons_lib`, and `reasons_lib.api`.
- [file] `reasons_lib/__init__.py` — The shim implementation; the place to add a `DeprecationWarning` before eventual removal.
- [file] `pyproject.toml` — Line 31 controls whether `reasons_lib` is included in built wheels; removing it here is the actual packaging change.
- [general] `rms-to-reasons rename history` — The `[0.3.0]` changelog says the project was renamed from `rms` to `reasons`, which may affect which old package/import names to search externally.

## Beliefs

- `reasons-lib-is-backwards-compatibility-shim` — `reasons_lib/__init__.py` contains only a compatibility docstring and `from reasons import *`.
- `pyproject-packages-reasons-lib` — `pyproject.toml` line 31 includes `"reasons_lib"` in the setuptools package list.
- `reasons-lib-has-no-internal-python-imports` — The Python grep for `reasons_lib` returned zero matches in repository Python files.
- `ftl-reasons-version-is-0.58.0` — `pyproject.toml` line 8 declares version `0.58.0`.
- `reasons-lib-api-was-documented-in-changelog` — `CHANGELOG.md` line 136 records an old Python API layer at `reasons_lib.api`.
