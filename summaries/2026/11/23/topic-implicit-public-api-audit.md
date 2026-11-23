---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Compare `dir(reasons)`, `dir(reasons_lib)`, and each `reasons_lib/*.py` shim against the intended documented API.

# Comparing `dir(reasons)`, `dir(reasons_lib)`, and the `reasons_lib/*.py` shims against the intended API

## What the docs say the API should be

Four references define the intended surface:

- `CHANGELOG.md:136` — "Python API layer (`reasons_lib.api`) returning dicts for CLI and tool-call consumers"
- `CLAUDE.md:24` — `reasons/` is importable as `import reasons`, *also aliased as `reasons_lib` for backwards compat*
- `CLAUDE.md:27` — `api.py` is the "Functional Python API (returns dicts…)"
- `CLAUDE.md:35` — `reasons_lib/` is a "backwards-compat shim — re-exports from `reasons/`"

So the intended contract is:

1. `import reasons` and `import reasons_lib` should *feel equivalent*.
2. `reasons_lib.api` (and `reasons.api`) should expose the full functional API: `init_db`, `add_node`, `add_justification`, `retract_node`, `assert_node`, `propagate`, `get_status`, `show_node`, `explain_node`, `add_nogood`, `get_belief_set`, `get_log`, `export_network`, `list_nodes`, `search`, `list_gated`, `list_negative`, `supersede`, `ensure_namespace`, `list_namespaces`, `clear_network_cache`.
3. Both packages should also expose the dataclasses `Justification`, `Node`, `Nogood` from `reasons/__init__.py`.

The tests in `tests/test_api.py` reinforce this: they import `from reasons import api`, `from reasons.storage import Storage`, `from reasons.api import _fts_search`, `from reasons.api import _fts_query`, and `from reasons.llm.invoke_model`.

---

## `dir(reasons)`

`reasons/__init__.py` (59 lines total) contains only the docstring and three `@dataclass` definitions; the imports at lines 3–4 are `from dataclasses import dataclass, field`.

Because there is **no `__all__`** anywhere in the canonical package (`canonical_module_export_lists` has `total_count: 0`), `dir(reasons)` will contain:

- Intended: `Justification`, `Node`, `Nogood`
- **Leaked**: `dataclass`, `field` (from the top-level import)
- Dunders (`__name__`, `__doc__`, `__path__`, etc.)
- **No submodules** by default — `api`, `storage`, `network`, `llm`, `pg`, `metadata`, `pubsub` are only present once something (e.g., `import reasons.api`) has caused Python to attach them to `reasons.__dict__`.

So `dir(reasons)` is **almost right**, but it never shows the submodules you actually need. `from reasons import api` still works because Python's import machinery auto-resolves `reasons.api` as a submodule when the name isn't found in `__dict__` — but that's an import-side magic trick, not an entry in `dir()`.

---

## `dir(reasons_lib)`

`reasons_lib/__init__.py` is three lines:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

Since `reasons` has **no `__all__`**, `from reasons import *` copies every non-underscore name from `reasons.__dict__` *at the moment the import runs*. Combined with the fact that `reasons/__init__.py` only defines the three dataclasses plus the two leaked imports, `dir(reasons_lib)` will contain:

- `Justification`, `Node`, `Nogood` ✓
- `dataclass`, `field` ✗ (leak)
- Dunders

It will **not** contain `api` unless `reasons.api` had already been imported at the time `reasons_lib` first loaded. There is no `__getattr__` fallback and no explicit `from reasons import api` — and `all_shim_export_patterns` confirms there is *no* `__getattr__` anywhere.

### The critical gap

`import reasons_lib; reasons_lib.api.add_node(...)` **fails**, because `api` is not an attribute of the `reasons_lib` package until the user explicitly does `from reasons_lib import api` (which then routes through the shim file `reasons_lib/api.py`). The docs (`CHANGELOG.md:136`) talk about `reasons_lib.api` as if it were addressable, so the shim package should import its submodule at `__init__` time (or use `__getattr__`) for the contract to hold.

---

## `dir(reasons_lib.api)` — the API-surface shim

`reasons_lib/api.py` is:

```python
"""Backwards-compatibility shim — use ``from reasons.api`` instead."""
from reasons.api import *  # noqa: F401,F403
```

`reasons/api.py` (5886 lines) also has **no `__all__`**, and its header (lines ~10–20) imports a large number of helper modules:

```
import copy, json, logging, os, re, sqlite3, sys
from collections.abc import Callable
from datetime import datetime, timezone
from itertools import combinations
from pathlib import Path
from . import Justification, pubsub
from .metadata import build_meta
from .network import Network
from .storage import PROPOSALS_SCHEMA, Storage
```

Therefore `dir(reasons_lib.api)` will contain the **entire public functional API** (init_db, add_node, retract_node, assert_node, propagate, get_status, show_node, explain_node, add_nogood, get_belief_set, get_log, export_network, list_nodes, search, list_gated, list_negative, supersede, ensure_namespace, list_namespaces, clear_network_cache, add_justification, DEFAULT_DB, SOURCE_TYPES) **plus**:

- All the leaked imports above: `copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Callable`, `datetime`, `timezone`, `combinations`, `Path`, `pubsub`, `build_meta`, `Network`, `PROPOSALS_SCHEMA`, `Storage`, `logger`
- The dataclasses `Justification`, `Node`, `Nogood` (because `reasons.api` re-imports `Justification` from `.`)

It will **not** contain (because star-import skips underscores without an `__all__`):

- `_fts_search`, `_fts_query` — the private helpers that `tests/test_api.py` explicitly imports from `reasons.api`. So `reasons_lib.api._fts_search` does not exist even though `reasons.api._fts_search` does. That's a behavioral asymmetry, though in practice it's probably fine that the shim drops private helpers.
- `_with_network`, `_pg_dispatch`, `_resolve_namespace`, `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_network_cache`

There is also **no corresponding `test_api.py` import path** through `reasons_lib.api`. Every test in `tests/test_api.py` imports `from reasons import api` and calls `api.<fn>` — none of them route through `reasons_lib.api`. So *if* the shim silently broke (e.g., a module rename), the test suite wouldn't catch it.

---

## Summary of divergences

| Surface | Documented as | Actually contains |
|---|---|---|
| `reasons` | Canonical package; `import reasons` | `Justification`, `Node`, `Nogood`, `dataclass`, `field` — **no submodules in `dir()`** |
| `reasons_lib` | Drop-in alias for `reasons` | Only the leaked top-level names — **no `api` attribute until explicitly imported** |
| `reasons_lib.api` | "Python API layer" (`CHANGELOG.md:136`) | All public functions from `reasons.api` **plus every top-level import leaked by `reasons.api`**; **minus `_`-prefixed helpers** |

The downstream consequence is that the "documented" API is not a frozen, testable surface:

- `dir(reasons)`, `dir(reasons_lib)`, and `dir(reasons_lib.api)` do **not** agree with each other.
- `import reasons_lib` does **not** imply `reasons_lib.api` is available (`AttributeError` until `from reasons_lib import api`).
- `reasons_lib.api` exposes a superset of the intended API (`copy`, `json`, `Storage`, `Network`, …) and a subset (no `_fts_search`). The only reason the tests still pass is that they import private helpers from the *canonical* module, not the shim.

## What's missing from the observations

- **Actual `dir()` output** — all of the above is derived from `__init__.py`/import statements, not from running `dir()` at runtime. Runtime `dir()` depends on import order (module-level attachment of submodules to the package).
- **Complete list of `reasons_lib/*.py` shims** — only `__init__.py` and `api.py` were provided. `tests/test_api.py` uses `from reasons.storage import Storage`, `from reasons.llm.invoke_model`, and `from reasons.api import _fts_search`, so the existence or absence of `reasons_lib/storage.py`, `reasons_lib/network.py`, `reasons_lib/llm.py`, `reasons_lib/pg.py`, `reasons_lib/metadata.py`, `reasons_lib/pubsub.py` shims matters for whether `import reasons_lib.*` works.
- **The `all_shim_export_patterns` search returned 0 matches** even though the files clearly contain `from reasons import *` and `from reasons.api import *`. That search result is inconsistent with the file contents — it may have been scoped to docstrings, run before the files were written, or matched on a different path. It should not be trusted as evidence that no shims exist.
- **The full 5886-line `reasons/api.py`** is not shown; only the first 500 lines. Any additional public (or accidentally public) names beyond that are unknown.

---

## Topics to Explore

- [file] `reasons/api.py` — The canonical functional API; confirm the full public surface and whether any `__all__` was later added to freeze it
- [file] `reasons_lib/__init__.py` — Check whether a `from reasons import api` or `__getattr__` was added so that `reasons_lib.api` is available after plain `import reasons_lib`
- [file] `tests/test_api.py` — Every test imports `from reasons import api` and never touches `reasons_lib.api`; adding at least one contract test against the shim would lock down the backwards-compat path
- [general] `dir-diff-audit` — Run `python -c "import reasons, reasons_lib, reasons_lib.api; import json; print(json.dumps({m: sorted(n for n in dir(eval(m)) if not n.startswith('_')) for m in ['reasons','reasons_lib','reasons_lib.api']}, indent=2))"` to see the actual sym-diff rather than inferring it
- [general] `shim-inventory` — Enumerate `reasons_lib/*.py` to determine whether `storage`, `network`, `llm`, `pg`, `metadata`, and `pubsub` shims exist, since `tests/test_api.py` bypasses the shim for all of them

## Beliefs

- `no-all-in-canonical-or-shim-modules` — Neither `reasons/__init__.py`, `reasons/api.py`, `reasons_lib/__init__.py`, nor `reasons_lib/api.py` defines `__all__`, so every public name exposed by these modules is decided implicitly by `from ... import *` semantics.
- `star-import-leaks-helper-modules` — Because `reasons/api.py` imports `copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Callable`, `datetime`, `Path`, `pubsub`, `build_meta`, `Network`, `PROPOSALS_SCHEMA`, and `Storage` at module scope, `dir(reasons_lib.api)` is a strict superset of the documented functional API.
- `reasons-lib-api-not-auto-imported` — `import reasons_lib` does not bind `reasons_lib.api`; only `from reasons_lib import api` (or `import reasons_lib.api`) resolves the shim, so `reasons_lib.api.add_node(...)` raises `AttributeError` after a bare `import reasons_lib`.
- `shim-drops-private-helpers` — `from reasons.api import *` excludes underscore-prefixed names, so `reasons_lib.api._fts_search` and `reasons_lib.api._fts_query` do not exist even though `reasons.api._fts_search` and `reasons.api._fts_query` do (and are used by `tests/test_api.py`).
- `tests-bypass-shim-path` — `tests/test_api.py` imports the API exclusively through `from reasons import api`, `from reasons.api import ...`, and `from reasons.storage import Storage`, so the `reasons_lib` backwards-compat path is effectively untested.
