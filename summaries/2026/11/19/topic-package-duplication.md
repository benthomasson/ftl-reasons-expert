---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Determine why `reasons_lib/` mirrors `reasons/` and whether it is legacy or still consumed.

## Explanation

`reasons_lib/` is not a second implementation of the package. The observed files show it is an **import-compatibility shim** over the canonical `reasons/` package.

Evidence:

- `reasons_lib/__init__.py:1-3` explicitly states:
  - `"Backwards-compatibility shim — use ``import reasons`` instead."`
  - then does `from reasons import *`.
- `reasons_lib/cli.py` similarly does `from reasons.cli import *`, so the old CLI import path is also re-exported.
- The canonical implementation is in `reasons/__init__.py:1-59`, which defines the core dataclasses `Justification`, `Node`, and `Nogood`.
- `pyproject.toml` entry points target the canonical package, not the shim:
  - `reasons = "reasons.cli:main"`
  - `reasons-mcp = "reasons.mcp_server:main"`
- However, `pyproject.toml` still packages both namespaces:
  - `packages = ["reasons", "reasons_lib"]`

So the mirror exists to preserve the old `reasons_lib` import path while directing new code to `reasons`.

### Is it legacy or still consumed?

It is **legacy in the compatibility sense**, but it is still **installed and distributed** because `pyproject.toml` includes `reasons_lib` in `[tool.setuptools].packages`.

Internal consumption is not visible: the provided search found **zero Python references to `reasons_lib`**, and no docs/changelog mention of `legacy`, `deprecated`, or `reasons_lib`. The regex for test imports of canonical `reasons` also found no matches, though that may be a limited search pattern.

Therefore, from the repository evidence we can say:

- `reasons` is the canonical package.
- `reasons_lib` is a backwards-compatibility alias.
- It is not consumed by observed internal Python code.
- It may still be consumed by external users, because it remains packaged and no removal/deprecation evidence appears in the observations.

We cannot prove external usage is zero without history, release notes, issue references, PyPI/download telemetry, or external consumer data. The full contents of `reasons_lib/` are also missing, so we can only confirm the observed files are shims.

## Topics to Explore

- [file] `reasons_lib/` — Inspect every file to confirm whether all modules are shims or whether any unique implementation remains.
- [file] `pyproject.toml` — Verify packaging and entry-point behavior if `reasons_lib` were removed or deprecated.
- [diff] `reasons_lib/__init__.py` vs `reasons/__init__.py` — Confirm the shim re-exports the canonical API rather than drifting from it.
- [general] `external-consumer-usage` — Search issues, PyPI metadata, telemetry, and downstream repos for actual `reasons_lib` imports.
- [general] `deprecation-policy` — Decide whether to keep the shim silently, add warnings, or schedule removal.

## Beliefs

- `reasons-lib-is-backwards-compatibility-shim` — `reasons_lib/__init__.py` declares itself a backwards-compatibility shim and re-exports `reasons` via `from reasons import *`.
- `reasons-is-canonical-implementation` — `reasons/__init__.py` defines the core data model and `pyproject.toml` entry points target `reasons.cli` and `reasons.mcp_server`.
- `reasons-lib-is-packaged` — `pyproject.toml` includes both `reasons` and `reasons_lib` in `[tool.setuptools].packages`.
- `no-internal-python-references-to-reasons-lib` — The provided Python-reference search found zero matches for `reasons_lib`.
- `reasons-lib-cli-reexports-reasons-cli` — `reasons_lib/cli.py` imports `*` from `reasons.cli`.
