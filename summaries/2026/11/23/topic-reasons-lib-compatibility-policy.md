---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Decide whether `reasons_lib` is still a supported external import path or can be removed as an internal rename cleanup.

# Decision: Is `reasons_lib` still supported, or safe to remove?

**Short answer: keep it — it is explicitly documented and packaged as an external backwards-compatibility path, not as an internal implementation detail. Removing it now is a breaking change, not an internal cleanup.**

## Evidence from the codebase

### It is documented as an intentional public alias
Two independent doc statements in `CLAUDE.md` make the intent unambiguous:

- `CLAUDE.md:24` — `reasons/ # importable as 'import reasons' (also aliased as reasons_lib for backwards compat)`
- `CLAUDE.md:35` — `reasons_lib/ # backwards-compat shim — re-exports from reasons/`

The word "backwards compat" appears in both the tree diagram and the directory annotation. This is not incidental.

`CHANGELOG.md:136` also references `reasons_lib.api` as the historical Python API surface:
> `- Python API layer (reasons_lib.api) returning dicts for CLI and tool-call consumers`

This indicates `reasons_lib.api` was the *documented public entry point* for external consumers (CLI + tool-call integrations), which is precisely the kind of path a backwards-compat shim is meant to preserve.

### It is packaged for distribution
`pyproject.toml:31`:
```toml
packages = ["reasons", "reasons_lib"]
```
The build system explicitly ships `reasons_lib` as an installable package. If it were merely an internal alias left over from a rename, it would normally be omitted from packaging (internal packages are not required at install time to be importable under both names). The fact that it is packaged means it is installed on end-user machines and importable from outside the repo.

### The shims are purpose-built re-exports
- `reasons_lib/__init__.py` — 3 lines, docstring *"Backwards-compatibility shim — use `import reasons` instead."*, then `from reasons import *`
- `reasons_lib/api.py` — 3 lines, docstring *"Backwards-compatibility shim — use `from reasons.api` instead."*, then `from reasons.api import *`

These are exactly the shape of an intentional compatibility layer, not accidental leftover code. Their docstrings name the intent directly.

### Internal Python never imports `reasons_lib`
`reasons_lib_mentions_in_python: total_count: 0` across `matches: []`.

This is the piece that *could* be misread as "cleanup is safe." But zero internal usage means only "internal modules have already migrated to `reasons`" — it says nothing about external consumers, who by definition are not in this codebase snapshot. The shim's whole purpose is to serve consumers that this repo cannot see.

## Decision

`reasons_lib` is **a supported external import path**, not an internal rename cleanup. It should be removed only under one of these conditions:

1. The project is bumping a major version and formally deprecating the `reasons_lib` namespace in release notes, *and*
2. There is evidence that no external consumers remain (download stats, downstream dependency checks, or a full deprecation cycle where `reasons_lib` imports **emit a `DeprecationWarning`** while still working).

Absent those, removal would break:
- Any consumer importing `from reasons_lib import Justification, Node, Nogood` (the three public dataclasses defined in `reasons/__init__.py:10`, `:27`, `:54`)
- Any consumer importing `from reasons_lib.api import ...` (the public dict-returning API layer)

## What's missing to make a firm "remove it" call

The observations cannot resolve the *external* question because they only cover this repository:

- **No external-consumer evidence**: no PyPI download stats, no downstream repo searches, no CI snapshot of downstream importers.
- **No deprecation history**: we cannot see whether `reasons_lib` was announced as deprecated in any prior release; `CHANGELOG.md:136` mentions it as an *added* API, not a deprecated one.
- **No `__all__` declarations captured**: `reasons/__init__.py` does not define `__all__`, so `from reasons import *` in `reasons_lib/__init__.py` will currently leak `dataclass` and `field` (both imported at `reasons/__init__.py:3`) into the `reasons_lib` namespace. This is a latent bug in the shim, but it complicates any "just remove it" decision because the re-export surface was never made minimal.
- **`reasons/api.py` contents not captured**: `reasons_lib/api.py` does `from reasons.api import *`, and we don't know whether `reasons/api.py` defines `__all__`, so the same leakage concern applies there.

## Topics to Explore

- [file] `reasons/api.py` — The shim's `from reasons.api import *` re-exports from here; check for `__all__` to confirm the shim's public surface is intentional.
- [file] `reasons/__init__.py` — Verify `Justification`, `Node`, and `Nogood` are the complete public API and whether adding `__all__` would tighten the shim.
- [general] `reasons-lib-deprecation-policy` — Search git history and prior CHANGELOG entries for a formal deprecation of `reasons_lib`; absence indicates it is still intended.
- [general] `downstream-consumer-audit` — Look for other repos or published packages importing `reasons_lib`; zero downstream usage plus a major version bump would make removal defensible.
- [diff] `reasons_lib-to-reasons-rename` — Inspect the commit that introduced `reasons_lib` to confirm it was an external-API alias rather than a purely internal temporary name.

## Beliefs

- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/__init__.py` and `reasons_lib/api.py` are documented as "Backwards-compatibility shim" in their module docstrings.
- `reasons-lib-is-packaged` — `pyproject.toml:31` includes `reasons_lib` in `[tool.setuptools] packages`, so it ships in the installed distribution.
- `reasons-lib-has-zero-internal-usage` — No Python files in the snapshot import `reasons_lib` (`reasons_lib_mentions_in_python.total_count == 0`).
- `reasons-lib-re-exports-reasons` — `reasons_lib/api.py` imports `*` from `reasons.api`, and `reasons_lib/__init__.py` imports `*` from `reasons`.
- `reasons-lib-documents-public-api` — `CHANGELOG.md:136` records `reasons_lib.api` as the Python API layer returning dicts for CLI and tool-call consumers.
- `reasons-init-lacks-__all__` — `reasons/__init__.py` does not declare `__all__`, so the wildcard re-export from `reasons_lib` may leak `dataclass` and `field`.
