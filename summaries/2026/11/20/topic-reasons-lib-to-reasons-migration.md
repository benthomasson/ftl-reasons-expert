---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Are the remaining `reasons_lib/*.py` files all shims like this one, or do some still hold real logic? Determines how far the rename has progressed and what is safe to delete.

## Short answer

Yes — the directly inspected `reasons_lib` modules are pure backward-compatibility shims. They contain only a docstring and a star import from the canonical `reasons` package.

| File | Lines | Real logic? | What it does |
|---|---:|---|---|
| `reasons_lib/__init__.py` | 3 | No | Line 2: `from reasons import *` |
| `reasons_lib/api.py` | 3 | No | Line 2: `from reasons.api import *` |
| `reasons_lib/storage.py` | 3 | No | Line 2: `from reasons.storage import *` |

The aggregate search for real definitions also supports this: `find_real_definitions_in_old_package` found **0** matches for `^(def|class) ` in `reasons_lib`. Even the likely substantive old module, `reasons_lib/storage.py`, is a 3-line shim.

So the rename appears largely complete: `reasons` is the canonical package, and `reasons_lib` has been reduced to an import facade.

## Important caveats

There are two inconsistencies in the observations that mean we should not overstate the evidence:

1. `find_new_package_imports_in_old_package` reports **0** matches for `^(from reasons|import reasons)`, but the inspected shim files literally contain:
   - `from reasons import *`
   - `from reasons.api import *`
   - `from reasons.storage import *`

   So that aggregate search result is not reliable as an exhaustive scan.

2. `find_consumers_of_old_package` reports **0** matches for `reasons_lib`, but `pyproject.toml` still contains:

   ```toml
   [tool.setuptools]
   packages = ["reasons", "reasons_lib"]
   ```

   So there is still at least one repo-level reference to `reasons_lib`.

The observations also do not list every file under `reasons_lib/*.py`, so we cannot prove “every remaining file” is a shim unless we enumerate the directory. However, based on the three inspected files, the zero `def`/`class` search, and the absence of `DeprecationWarning`, `warnings.warn`, `__getattr__`, and `__all__`, it is very likely that the whole package is now shim-only.

## Is it safe to delete?

Internally, probably yes — assuming the zero-consumer search is accurate for Python code and tests. The canonical package is `reasons`, and the console scripts already point at it:

```toml
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

But it is **not safe to delete unconditionally** because:

- `reasons_lib` is still explicitly packaged in `pyproject.toml`.
- This package exists for backward compatibility, so external users may still import `reasons_lib` even if the repo has no internal consumers.
- The shims do not emit deprecation warnings, so deleting them would be a silent breaking change for external users.

Recommended sequence:

1. Enumerate all remaining files under `reasons_lib/`.
2. Confirm no repo code imports `reasons_lib`.
3. If external compatibility still matters, add deprecation warnings and keep the shims for a release cycle.
4. Remove `"reasons_lib"` from `[tool.setuptools].packages`.
5. Then delete the directory.

## Topics to Explore

- [file] `reasons_lib/` — Enumerate every remaining `.py` file to confirm whether any file is more than a 3-line star-import shim.
- [file] `pyproject.toml` — Update `[tool.setuptools].packages` before deleting `reasons_lib`, otherwise the package metadata will still reference it.
- [general] `reasons-lib-compatibility-policy` — Decide whether `reasons_lib` is still a supported external import path or can be removed as an internal rename cleanup.
- [diff] `reasons_lib-to-reasons-rename` — Compare the old shim modules against `reasons/__init__.py`, `reasons/api.py`, and `reasons/storage.py` to verify re-export parity.
- [general] `external-consumer-search` — Search outside the repo, such as downstream packages or documentation, for `import reasons_lib` and `from reasons_lib`.

## Beliefs

- `reasons_lib-is-a-shim-package` — `reasons_lib/__init__.py`, `reasons_lib/api.py`, and `reasons_lib/storage.py` are 3-line star-import shims pointing at `reasons`.
- `reasons_lib-has-no-real-definitions` — The aggregate `^(def|class) ` search across `reasons_lib` returns zero matches.
- `reasons_lib-requires-pyproject-update-before-deletion` — `pyproject.toml` still lists `reasons_lib` in `[tool.setuptools].packages`.
- `reasons_lib-does-not-emit-deprecation-warnings` — No `DeprecationWarning`, `warnings.warn`, `__getattr__`, or `__all__` markers were found in `reasons_lib`.
- `reasons_lib-api-maps-to-reasons-api` — `reasons_lib/api.py` re-exports `reasons.api` via `from reasons.api import *`.
