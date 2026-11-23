---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search the repo and published packages for `import reasons_lib` and `from reasons_lib` to decide whether wholesale deletion breaks callers.

## Explanation

I’d separate **repo-internal deletion safety** from **published-package deletion safety**.

### Repo-internal callers

The repo scan found **zero Python references** to `reasons_lib`:

- `direct_import_statements`: 0 matches for `import reasons_lib`
- `direct_from_import_statements`: 0 matches for `from reasons_lib`
- `all_python_reasons_lib_references`: 0 matches for any `reasons_lib` reference in Python files

So, as far as the observed repository code is concerned, there are **no internal Python callers** that would immediately break if `reasons_lib/` were deleted.

The only references are documentation/commentary:

- `CLAUDE.md:24` says the `reasons/` directory is importable as `import reasons` and “also aliased as reasons_lib for backwards compat”
- `CLAUDE.md:35` describes `reasons_lib/` as a “backwards-compat shim — re-exports from reasons/”
- `CHANGELOG.md:136` mentions a historical Python API layer `reasons_lib.api`

### Package metadata

`pyproject.toml:31` explicitly includes the compatibility package:

```toml
packages = ["reasons", "reasons_lib"]
```

That means the published `ftl-reasons` package is configured to ship `reasons_lib`. The shim itself is tiny: `reasons_lib/__init__.py:2` does:

```python
from reasons import *  # noqa: F401,F403
```

The CLI entry points do **not** depend on `reasons_lib`; `pyproject.toml` maps `reasons` to `reasons.cli:main` and `reasons-mcp` to `reasons.mcp_server:main`.

### Decision

For **repo-internal code**, wholesale deletion appears safe because there are no Python imports.

For **published-package callers**, deletion is a potential breaking change. The package is explicitly configured to include `reasons_lib`, and the repo docs describe it as a backwards-compatibility shim. External consumers may still use `import reasons_lib` or `from reasons_lib import ...` even though this repo no longer does.

The observations do **not** include an actual scan of published wheel/sdist contents or downstream consumers, so they are insufficient to prove that no external caller exists. If the project intends to remove `reasons_lib`, it should either deprecate it across a release cycle or treat removal as a major-version breaking change. Also, if `reasons_lib/` is deleted, `pyproject.toml:31` must be updated; otherwise setuptools package metadata will be inconsistent.

## Topics to Explore

- [general] `published-wheel-contents` — Inspect built wheels/sdists for `ftl-reasons` to confirm what `reasons_lib` files actually ship and whether submodules like `reasons_lib.api` exist.
- [general] `downstream-import-scan` — Search dependent repositories or package usage data for `import reasons_lib` / `from reasons_lib` before treating deletion as safe.
- [file] `pyproject.toml` — Line 31 lists `reasons_lib` as a packaged module; this must change if the shim is removed.
- [file] `reasons_lib/__init__.py` — The only observed shim implementation; it re-exports `reasons` top-level names.
- [file] `CHANGELOG.md` — Line 136 mentions historical `reasons_lib.api`; check whether the public compatibility surface was ever broader than the current shim.

## Beliefs

- `reasons-lib-has-no-in-repo-python-importers` — The observed repo contains zero Python references to `reasons_lib`, including zero `import reasons_lib` and zero `from reasons_lib` statements.
- `reasons-lib-is-packaged-in-ftl-reasons` — `pyproject.toml:31` includes `"reasons_lib"` in `[tool.setuptools] packages`, so published distributions include it unless packaging changes.
- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/__init__.py:2` re-exports `reasons` via `from reasons import *`.
- `cli-entry-points-do-not-use-reasons-lib` — `[project.scripts]` maps commands to `reasons.cli:main` and `reasons.mcp_server:main`, not to `reasons_lib`.
- `internal-scan-cannot-prove-external-safety` — The observations cover repo/package metadata but not published package contents or downstream consumers, so lack of internal imports does not prove external callers are absent.
