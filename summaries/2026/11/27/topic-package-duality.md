---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Why `reasons/` and `reasons_lib/` are near-identical, which is canonical, and how imports cross between them (determines everything downstream).

## Short answer

`reasons/` is canonical. `reasons_lib/` is a backwards-compatibility facade with the same module layout, but it does not contain the implementation. Its files are thin shims that re-export from `reasons/`.

The dependency is intentionally one-way:

```text
reasons_lib  ->  reasons
```

Code in `reasons/` should not import `reasons_lib/`. That one-way flow is what keeps the compatibility layer from becoming a circular dependency.

## Why they look “near-identical”

They are near-identical in **module names and directory shape**, not in implementation.

Examples from the observations:

- `reasons_lib/__init__.py:1-2` says it is a backwards-compatibility shim and does `from reasons import *`.
- `reasons_lib/api.py:1-2` re-exports `from reasons.api import *`.
- `reasons_lib/cli.py:1-2` re-exports `from reasons.cli import *`.
- The same pattern appears for `ask.py`, `build_wiki.py`, `check_stale.py`, `cluster.py`, `compact.py`, `contradictions.py`, `derive.py`, `export_card.py`, `export_markdown.py`, `hf.py`, `import_agent.py`, etc.

So an old import like:

```python
from reasons_lib.api import something
```

resolves through a shim into:

```python
from reasons.api import *
```

The shim files are intentionally tiny. The “sameness” is there so old package paths keep working, not because there are two real implementations.

## Which package is canonical?

`reasons/` is canonical. Evidence:

- `reasons/__init__.py` contains the actual dataclasses: `Justification`, `Node`, and `Nogood`.
- `reasons_lib/__init__.py:2` only does `from reasons import *`.
- `pyproject.toml` defines console scripts against canonical modules:
  - `reasons = "reasons.cli:main"`
  - `reasons-mcp = "reasons.mcp_server:main"`
- `CLAUDE.md:24` says `reasons/` is importable as `import reasons`, “also aliased as reasons_lib for backwards compat”.
- `CLAUDE.md:35` says `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/”.
- `CHANGELOG.md:136` references `reasons_lib.api` as an older Python API layer, consistent with a historical rename or compatibility transition.

`reasons_lib/` is distributed, but it is not the source of truth.

## How imports cross between them

The compatibility layer imports from canonical code:

- `reasons_lib/__init__.py:2`: `from reasons import *`
- `reasons_lib/api.py:2`: `from reasons.api import *`
- `reasons_lib/cli.py:2`: `from reasons.cli import *`
- Same pattern across the other observed shim files.

Canonical code imports canonical modules directly:

- `reasons/mcp_server.py:23`: `from reasons import api`
- `reasons/pg.py:943`: `from reasons.api import _STOP_WORDS`
- `reasons/pg.py:1324`: `from reasons.api import _TOPIC_STOP_WORDS`
- `reasons/cli.py` imports `api` via a relative-style import, based on the observed `from_imports` entry with module `""` and name `api`.

There is no observed `reasons/` module importing `reasons_lib/`. That is the key architectural invariant: **canonical code must not depend on the shim**. If `reasons/__init__.py` imported `reasons_lib`, the shim’s `from reasons import *` could create circular-import failures during package initialization.

## Downstream consequences

Because the shims use `from ... import *`, downstream behavior has some important details:

- Re-exported classes/functions are usually the same Python objects. For example, if `Node` is star-imported from `reasons`, then `reasons_lib.Node` should be the same class object as `reasons.Node`. `isinstance()` checks across the two import paths should therefore interoperate.
- The module objects are still distinct. `reasons_lib.api` is a separate module from `reasons.api`; it merely re-exports names.
- Monkeypatching differs. Rebinding `reasons_lib.api.some_name` does not mutate `reasons.api.some_name`. Mutating a shared object would, but rebinding a shim attribute is not equivalent to changing canonical module state.
- Private names are not generally available through `import *`. Canonical internal code imports private names directly from `reasons.api` (`reasons/pg.py:943,1324`), not through `reasons_lib`.
- New canonical modules do not automatically become importable as `reasons_lib.<module>` unless a shim file is added. The shim tree must be maintained for the backwards-compatible public surface.
- Entry points always target canonical modules. Running the CLI uses `reasons.cli:main`, not `reasons_lib.cli:main`.

## Caveats / missing information

The observation named `all_reasons_lib_references` reports zero matches, while the broader import scan lists `reasons_lib` shim files. That is likely a query-scope difference. If the zero-match scan was for internal imports of `reasons_lib` from canonical code, it supports the conclusion that current code does not depend on the shim. However, the dataset also clearly lists shim files that reference `reasons`.

Also, the observations do not show whether canonical modules define `__all__`. Without `__all__`, `from reasons import *` also re-exports public imported names from `reasons/__init__.py`, such as `dataclass` and `field`. The full shim inventory is truncated, so we cannot confirm every canonical module has a matching shim.

## Topics to Explore

- [file] `reasons/api.py` — Canonical API implementation behind `reasons_lib/api.py`; inspect its public/private surface and whether it defines `__all__`.
- [file] `reasons_lib/api.py` — Representative shim file; confirm it contains only the docstring and `from reasons.api import *`.
- [file] `pyproject.toml` — Packaging and entry-point configuration; confirm both packages are installed and only `reasons.*` is used for scripts.
- [general] `reasons_lib-shim-inventory` — Enumerate all `reasons_lib/` modules to see which canonical modules have compatibility shims and which do not.
- [general] `import-identity-and-monkeypatching` — Test whether `reasons_lib.Node is reasons.Node` and whether rebinding attributes on `reasons_lib.api` affects `reasons.api`.

## Beliefs

- `reasons-is-canonical-implementation` — `reasons/__init__.py` defines the real `Justification`, `Node`, and `Nogood` dataclasses, while `reasons_lib/__init__.py` only re-exports from `reasons`.
- `reasons-lib-requires-reasons` — Observed `reasons_lib` modules import from their `reasons` counterparts, and no observed `reasons` module imports `reasons_lib`.
- `entrypoints-target-reasons-only` — `pyproject.toml` maps `reasons` and `reasons-mcp` console scripts to `reasons.cli:main` and `reasons.mcp_server:main`, not to `reasons_lib`.
- `both-packages-are-packaged` — `[tool.setuptools] packages = ["reasons", "reasons_lib"]` causes the compatibility shim package to be distributed alongside canonical `reasons`.
- `shim-star-imports-limit-private-names` — `reasons_lib` shims use `from reasons... import *`, so private names are not re-exported through the shim, and canonical code imports private names directly from `reasons.api` in `reasons/pg.py:943,1324`.
