---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Reconcile the two packages: diff `reasons/` against `reasons_lib/` to confirm whether one is vendored legacy, and which is authoritative.

## Short answer

`reasons/` is authoritative. `reasons_lib/` is not a vendored legacy copy of the implementation; it is a backwards-compatibility facade that re-exports from `reasons/`.

Real behavior should be changed in `reasons/`. `reasons_lib/` exists so older external imports such as `reasons_lib.api` or `reasons_lib.cli` continue to resolve.

## What the diff shows

A full line-by-line diff is not possible from the supplied observations because we only have the full contents of `reasons/__init__.py` and `reasons_lib/__init__.py`, plus shim snippets for several `reasons_lib` modules. But the evidence is enough to reconcile direction:

| Package | Role | Evidence |
|---|---|---|
| `reasons/` | Real implementation | `reasons/__init__.py` defines `Justification`, `Node`, and `Nogood` dataclasses; `reasons/api.py` contains implementation well past line 3919; `reasons/cli.py` contains CLI behavior through at least line 3854. |
| `reasons_lib/` | Compatibility shim | `reasons_lib/__init__.py:1-2` is a docstring saying “Backwards-compatibility shim” followed by `from reasons import *`; submodule shims follow the same pattern, e.g. `reasons_lib/api.py:1-2`, `reasons_lib/ask.py:1-2`, `reasons_lib/cli.py:1-2`. |

Key reconciliation points:

1. **`reasons_lib` imports from `reasons`, not the other way around.**  
   `reasons_lib/__init__.py:2` does `from reasons import *`. `reasons_lib/api.py:2` does `from reasons.api import *`. This is the opposite of vendoring: a vendored legacy copy would contain an independent implementation and would not simply re-export the newer package.

2. **The package configuration intentionally installs both packages.**  
   `pyproject.toml:31` has `packages = ["reasons", "reasons_lib"]`. The console scripts point to `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib.*`, so the entry points exercise the authoritative package.

3. **Repo-internal code imports `reasons`, not `reasons_lib`.**  
   Examples include `reasons/mcp_server.py:23` (`from reasons import api`), `reasons/pg.py:943` and `:1324` (`from reasons.api import ...`), and `tests/conftest.py:22` (`from reasons.pg import PgApi`). The observation for `reasons_lib` import usages reports `total_count: 0`, so there are no in-repo consumers of the compatibility package.

4. **Repo guidance says `reasons_lib` is a shim.**  
   `CLAUDE.md:24` says `reasons/` is importable as `import reasons` and “also aliased as reasons_lib for backwards compat”; `CLAUDE.md:35` explicitly calls `reasons_lib/` a “backwards-compat shim — re-exports from reasons/”.

5. **Documentation references to `reasons_lib` are historical/generated, not runtime authority.**  
   `CHANGELOG.md:136` mentions `reasons_lib.api` as part of the Python API layer, and `.forge/proposed-entries.json:2-14` lists generated entries for `reasons_lib` modules. Those do not change the fact that the current implementation lives in `reasons/` and the shim files delegate to it.

6. **`reasons_lib` is not “vendored legacy.”**  
   A vendored legacy package would normally be a copied implementation with no dependency on the current package. Here, the shim files are tiny wrappers with docstrings like “Backwards-compatibility shim — use `from reasons.api` instead.” They cannot be the source of truth because they contain no implementation.

## Practical conclusion

Treat `reasons/` as the authoritative package. Treat `reasons_lib/` as an installed compatibility surface. If you need to add or change behavior, do it in `reasons/` first. Only touch `reasons_lib/` when you need to preserve compatibility for external imports, typically by adding a new wildcard shim for a newly exposed `reasons` submodule.

One maintenance caveat: the shims use wildcard imports. That means private names are not re-exported by `from reasons.api import *` unless the source module’s `__all__` or name set exposes them. Since there are no in-repo `reasons_lib` imports, CI will not catch compatibility gaps for private or unusual external imports.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the exact shim docstring and wildcard re-export behavior, including whether `__all__` is defined.
- [diff] `reasons/ vs reasons_lib/` — Run a full file-by-file diff to verify every `reasons_lib/*.py` file is only a shim and not an accidental stale implementation.
- [function] `reasons/cli.py:main` — This is the authoritative CLI entry point referenced by `pyproject.toml`, not a `reasons_lib` entry point.
- [file] `pyproject.toml` — Review packaging and console-script wiring to confirm `reasons_lib` is shipped only for compatibility.
- [general] `reasons_lib deprecation strategy` — Determine whether external consumers still need the shim and whether it can be documented or scheduled for removal.

## Beliefs

- `reasons-is-authoritative-over-reasons_lib` — `reasons/` contains the real implementation; `reasons_lib/` re-exports from it.
- `reasons_lib-requires-reasons` — `reasons_lib/__init__.py:2` and submodule shims import from `reasons`, so `reasons_lib` cannot function without `reasons`.
- `reasons_lib-is-a-backwards-compat-shim` — `CLAUDE.md:35` and shim docstrings identify `reasons_lib/` as a compatibility layer, not a vendored legacy package.
- `repo-code-depends-on-reasons-not-reasons_lib` — Internal imports and tests use `reasons.*`; the observation reports zero `reasons_lib` import usages in the repo.
- `pyproject-installs-both-packages` — `pyproject.toml:31` packages both `reasons` and `reasons_lib`, while console scripts target `reasons.cli:main` and `reasons.mcp_server:main`.
