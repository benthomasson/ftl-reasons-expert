---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The distribution must be installed with the name `ftl-reasons` for the real version path to work.

## Explanation

The codebase has two distinct names for the same project, and the version-reporting code depends on the *distribution* name — not the *import* name.

### The two names

**Distribution name** — the name a package is registered under in installed metadata (what `pip` sees):

```toml
# pyproject.toml (lines returned from start)
[project]
name = "ftl-reasons"
version = "0.58.0"
```

**Import name** — the directories that actually get imported in Python:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So the code you write is `import reasons`, but the *installed thing* is called `ftl-reasons`. These are decoupled. `importlib.metadata` resolves by **distribution name**, not import name.

### Where the version is read

Two places read the version at runtime, and both hardcode the distribution name `ftl-reasons`:

1. **`reasons/cli.py:3057`**
   ```python
   parser.add_argument("--version", action="version",
                       version=f"%(prog)s {_pkg_version('ftl-reasons')}")
   ```
   with the import at `reasons/cli.py:13`:
   ```python
   from importlib.metadata import version as _pkg_version
   ```

2. **`reasons/metadata.py:17-18`**
   ```python
   from importlib.metadata import version
   v = version("ftl-reasons")
   ```
   This call is at **module scope**, so it executes as soon as `reasons.metadata` is imported.

There is **no `__version__`** attribute anywhere (`__version__` search returned 0 matches), so there's no fallback and no independent source of truth inside the package. Every version-reporting code path funnels through `importlib.metadata` → `"ftl-reasons"`.

### Why the distribution must literally be named `ftl-reasons`

`importlib.metadata.version("ftl-reasons")` walks the installed `*.dist-info` / `*.egg-info` metadata directories and matches on the *declared distribution name* (PEP 503-normalized). It does not import anything and has no idea that `reasons/` is the importable package.

Consequences:

- If the project is **not installed** at all (e.g., running `python -m reasons.cli` from a checkout with only `PYTHONPATH`), no dist-info exists, and both call sites raise `PackageNotFoundError`.
- If the project is installed under a **different name** (a fork, a rename, vendoring), the lookup for `ftl-reasons` still fails for the same reason: the installed metadata name won't match.
- Because `reasons/cli.py:3057` resolves `_pkg_version('ftl-reasons')` **eagerly at parser-construction time**, a failure there prevents the CLI from even building its `argparse` parser — not just a failure when `--version` is passed. The same eager behavior applies at `reasons/metadata.py:18`, which fails at import.

So "the distribution must be installed with the name `ftl-reasons`" is a hard coupling between `pyproject.toml`'s `[project].name` and the string literals in `reasons/cli.py` and `reasons/metadata.py`. The `[project.scripts]` entries (`reasons`, `reasons-mcp`) also depend on the import name landing correctly, but only the version lookups depend on the *distribution* name matching.

### Not shown / gaps

- No `try/except PackageNotFoundError` is visible at either call site, so a fallback (e.g., `"unknown"` or a hardcoded string) is not demonstrated by these observations.
- Whether `reasons/metadata.py` is imported eagerly by `reasons.cli` (which would turn `--version` failures into CLI-startup failures) isn't confirmed by the snippets provided.

## Topics to Explore

- [file] `reasons/metadata.py` — Contains the module-scope `version("ftl-reasons")` call; check import ordering and any fallback.
- [function] `reasons/cli.py:main` — Watch whether `import reasons.metadata` occurs before or during parser construction, and whether a fallback wraps the lookup.
- [file] `pyproject.toml` — The source of truth for `[project].name`, `version`, `[tool.setuptools].packages`, and `[project.scripts]`; small file, worth reading whole.
- [general] `packaging-distribution-vs-import-name` — Clarifies `importlib.metadata` name matching vs. importable modules, and the PEP 503 normalization rules that make `ftl-reasons` and `ftl_reasons` equivalent.
- [general] `PackageNotFoundError-behavior` — Understand exactly when `importlib.metadata.version` raises vs. returns, especially in editable installs and zipapps.

## Beliefs

- `version-lookup-requires-ftl-reasons-distribution` — `importlib.metadata.version("ftl-reasons")` succeeds only when a distribution named `ftl-reasons` is installed.
- `pyproject-declares-distribution-name-ftl-reasons` — `pyproject.toml` sets `[project].name = "ftl-reasons"` with version `0.58.0`.
- `import-names-differ-from-distribution-name` — The packages `reasons` and `reasons_lib` are importable, but the distribution is registered as `ftl-reasons`.
- `cli-version-flag-eagerly-resolves-metadata` — `reasons/cli.py:3057` calls `_pkg_version('ftl-reasons')` when the argument parser is constructed, not when `--version` is invoked.
- `metadata-module-eagerly-resolves-version` — `reasons/metadata.py:18` calls `version("ftl-reasons")` at module import time.
- `codebase-has-no-__version__-fallback` — No `__version__` attribute is defined, so version reporting depends exclusively on installed distribution metadata.
