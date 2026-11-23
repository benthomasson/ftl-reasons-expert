---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Run a complete search for `reasons_lib`, including dynamic imports, docs, scripts, and downstream consumers; the displayed searches are truncated.

## Explanation

### What the search actually covered

The observation bundle runs several searches whose scopes differ, and reconciling them is the key to this explanation:

| Search | Pattern | Total |
|---|---|---|
| `all_reasons_lib_references` | `reasons_lib` | **0** |
| `python_import_statements` | `from reasons_lib\|import reasons_lib` | **0** |
| `dynamic_import_mechanisms` | `importlib\|import_module\|__import__\|find_spec\|pkgutil\|entry_points` | **5** |
| `markdown_documentation_mentions` | `reasons_lib` | **3** |
| `workflow_and_ci_mentions` | `reasons_lib` | **0** |

**First red flag:** the generic `all_reasons_lib_references` search returned `0`, yet `pyproject.toml`, `reasons_lib/__init__.py`, and three markdown files all literally contain `reasons_lib`. That means this generic search was either scoped to a file type that excludes `pyproject.toml`/`__init__.py`/`.md`, or it was truncated/limited. Treat its `0` as unreliable and rely on the targeted searches instead.

### What `reasons_lib` is

`reasons_lib/__init__.py` is a 3-line shim (observed at `reasons_lib/__init__.py:1-3`):

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *
```

So `reasons_lib` is **not a library of its own** — it is a namespace that re-exports everything from the real package `reasons` via a star import. This is confirmed by `CLAUDE.md:35` ("backwards-compat shim — re-exports from reasons/") and `CLAUDE.md:24` ("also aliased as reasons_lib for backwards compat").

### Packaging and console scripts

From `pyproject.toml`:

- `[project].name = "ftl-reasons"` (distribution name is **not** `reasons_lib`)
- `[project.scripts]` exposes only `reasons = "reasons.cli:main"` and `reasons-mcp = "reasons.mcp_server:main"` — **no entry point targets `reasons_lib`**.
- `[tool.setuptools] packages = ["reasons", "reasons_lib"]` — the shim *is* shipped on install, which is the only mechanism by which downstream consumers could still see it.

### Dynamic-import audit

The five hits under `dynamic_import_mechanisms` do **not** resolve `reasons_lib`:

- `reasons/cli.py:13` — `from importlib.metadata import version as _pkg_version` (reads installed version metadata, not a module).
- `reasons/metadata.py:17` — same `version` helper.
- `tests/test_pg_dispatch.py:43,59,85` — `__import__("os")` used to read environment variables.

There is no `import_module("reasons_lib")`, no `find_spec("reasons_lib")`, no `pkgutil` scan, and no plugin/entry-point loader that could unwrap `reasons_lib` at runtime. So the shim cannot be silently exercised through dynamic loading.

### No active Python importers

`python_import_statements` returned **0** matches for `from reasons_lib` / `import reasons_lib`. Combined with a CI search returning **0** mentions, this means:

- No module in the repo imports `reasons_lib`.
- No test imports it (so it's effectively untested for regression).
- No workflow or script references it.

The only remaining places `reasons_lib` appears are:
1. `pyproject.toml` (packaging list)
2. `reasons_lib/__init__.py` itself
3. Documentation: `CHANGELOG.md:136`, `CLAUDE.md:24`, `CLAUDE.md:35`

### A stale doc reference

`CHANGELOG.md:136` mentions `reasons_lib.api` — a submodule returning dicts for CLI/tool-call consumers. But the only file under `reasons_lib/` in these observations is `__init__.py`. So `reasons_lib.api` **no longer exists** as a module (or was consolidated into `reasons`), making that changelog entry a dangling reference. This is worth verifying before trusting external consumers who may still import `reasons_lib.api`.

### What's missing to close the search

The observations do not include:
- An actual `ls reasons_lib/` / recursive tree to confirm `__init__.py` is the only file.
- Any search of **outside** this repo (e.g., PyPI reverse dependencies, sibling projects) — the "downstream consumers" part of the task cannot be answered from this data.
- The regex used for `all_reasons_lib_references`, which would explain its incorrect `0`.
- Git history to know when `reasons_lib/api.py` was removed and whether the shim still needs to exist for a published version.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm it is the only file in the shim and re-exports are complete/current.
- [file] `pyproject.toml` — Verify whether `reasons_lib` should remain in `[tool.setuptools] packages` or be dropped as dead weight.
- [file] `CHANGELOG.md` — Investigate the `reasons_lib.api` reference at line 136 to determine whether that submodule ever shipped and when it was retired.
- [general] `downstream-consumers` — Search PyPI reverse dependencies and any sibling repos for `ftl-reasons` consumers importing `reasons_lib`.
- [general] `deprecation-removal-plan` — Decide whether to keep, formally deprecate with `DeprecationWarning`, or delete the shim in the next release.

## Beliefs

- `all-reasons-lib-references-is-zero` — The generic `reasons_lib` search returned 0 matches despite three documented markdown hits and the shim file itself, so its scope is inconsistent with the other searches.
- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` is a 3-line module whose only logic is `from reasons import *`.
- `reasons-lib-has-no-active-python-importers` — No `import reasons_lib` or `from reasons_lib` statements exist anywhere in the repository.
- `reasons-lib-has-no-dynamic-import-resolvers` — None of the five `importlib`/`__import__`/`find_spec`/`pkgutil` hits reference `reasons_lib`.
- `reasons-lib-is-packaged-via-setuptools` — `pyproject.toml` `[tool.setuptools] packages` lists `reasons_lib`, so the shim ships even though nothing in-repo imports it.
- `reasons-lib-api-reference-is-dangling` — `CHANGELOG.md:136` refers to `reasons_lib.api`, but no such submodule appears in the observed tree.
- `reasons-lib-has-no-ci-coverage` — All workflow/CI file searches for `reasons_lib` returned zero matches.
