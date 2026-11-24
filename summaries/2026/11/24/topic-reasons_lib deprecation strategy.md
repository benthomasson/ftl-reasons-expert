---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Determine whether external consumers still need the shim and whether it can be documented or scheduled for removal.

## Assessment

**Internal consumers no longer need the shim.** The gathered Python grep for `reasons_lib` returned `total_count: 0`, so no Python code in the repository imports it. Both observed shim files are pure re-exports:

- `reasons_lib/__init__.py:2` — `from reasons import *`
- `reasons_lib/api.py:2` — `from reasons.api import *`

They contain no independent logic and exist only to preserve old import paths.

**External consumers are not determinable from the repository alone.** The shim is still shipped: `pyproject.toml:31` includes `reasons_lib` in `[tool.setuptools].packages`. Historical docs also show it was a public surface: `CHANGELOG.md:136` documents `reasons_lib.api` as the Python API layer in `0.1.0`, and `CLAUDE.md:24` / `CLAUDE.md:35` explicitly describe `reasons_lib` as a backwards-compat alias/shim. That makes external usage plausible, but zero internal imports does not prove zero external imports.

**Recommendation:** document the shim as deprecated now, but do not remove it until external usage is checked and a deprecation window has passed. A safe sequence is:

1. Add an explicit deprecation note in `CHANGELOG.md` and update `CLAUDE.md:24` / `CLAUDE.md:35`.
2. Optionally emit `DeprecationWarning` from `reasons_lib/__init__.py` and `reasons_lib/api.py`.
3. Search external usage via GitHub code search, registry/dependent metadata, and issue reports.
4. Schedule removal for a later minor/major release if no external dependence appears.
5. On removal, update `pyproject.toml:31`, delete the `reasons_lib` package, update docs, and regenerate/remove the `.forge/proposed-entries.json` entries for `reasons_lib-*` files.

The observations do not include external usage data, PyPI download stats, downstream dependent searches, or README/package-registry metadata. They also only show the contents of two shim files, while `.forge/proposed-entries.json` lists additional `reasons_lib-*` documentation entries, so an inventory of all `reasons_lib` modules is needed before actual removal.

## Topics to Explore

- [file] `pyproject.toml` — Verify the exact packaging change required to stop shipping `reasons_lib`, especially line 31.
- [file] `reasons_lib/api.py` — Check whether a deprecation warning can be added without breaking wildcard re-exports.
- [general] `external-usage-search` — Search GitHub and package-registry dependents for `import reasons_lib` / `from reasons_lib`.
- [file] `CHANGELOG.md` — Add a deprecation/removal entry and inspect the historical `0.1.0` and `0.3.0` context.
- [file] `.forge/proposed-entries.json` — Regenerate or remove stale `reasons_lib-*` generated documentation entries if the shim is removed.

## Beliefs

- `reasons-lib-has-no-internal-python-imports` — The gathered Python grep for `reasons_lib` returned zero matches.
- `reasons-lib-is-included-in-setuptools-packages` — `pyproject.toml:31` lists `reasons_lib` alongside `reasons` in `[tool.setuptools].packages`.
- `reasons-lib-api-was-public-in-0.1.0` — `CHANGELOG.md:136` documents `reasons_lib.api` as the Python API layer for CLI and tool-call consumers.
- `reasons-lib-reexports-reasons` — `reasons_lib/__init__.py:2` and `reasons_lib/api.py:2` only wildcard re-export from `reasons` and `reasons.api`.
- `reasons-lib-docs-mark-it-backwards-compat` — `CLAUDE.md:24` and `CLAUDE.md:35` describe `reasons_lib` as a backwards-compat alias/shim.
