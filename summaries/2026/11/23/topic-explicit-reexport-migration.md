---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Plan the migration from wildcard shims to explicit imports and/or `__all__` while preserving backwards compatibility.

## Current state

`reasons_lib` is a compatibility facade over the real package, `reasons`. Every shim module currently re-exports via wildcard:

- `reasons_lib/__init__.py:2` — `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py:2` — `from reasons.api import *  # noqa: F401,F403`
- Same pattern appears in the other listed `reasons_lib/*.py` modules, all on line 2.

The real package has no explicit export contract:

- `explicit_all_definitions.total_count = 0`, so no module declares `__all__`.
- `reasons/__init__.py` imports `dataclass` and `field` at line 3 and defines `Justification`, `Node`, and `Nogood`. Because there is no `__all__`, `from reasons import *` currently exposes `dataclass`, `field`, `Justification`, `Node`, and `Nogood`.
- `reasons/api.py` is very large: 5,886 lines. The first 200 lines show public constants/functions such as `DEFAULT_DB`, `clear_network_cache`, and `init_db`, plus imported names like `json`, `sqlite3`, `Justification`, `pubsub`, `Network`, and `Storage`. Those imported names are currently re-exported by `from reasons.api import *` unless prefixed with `_`.

Packaging and entry points are relevant but lower risk:

- `pyproject.toml:31` packages both `reasons` and `reasons_lib`, so the shims ship in the distribution.
- `pyproject.toml:27-28` points console scripts at `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib`, so CLI behavior should not depend on the shim migration.

Important observation gap: the wildcard scan reports `total_count: 33`, but the provided `matches` list contains 30 entries. The first migration step must re-run the search to find the missing three. Also, `test_import_usage.total_count = 0`, which means the provided scan found no existing test imports of `reasons` or `reasons_lib`; compatibility coverage likely needs to be added.

## Migration plan

### 1. Freeze the current public surface before editing

Do not start by deleting wildcard imports. First capture exactly what is currently exported by every `reasons` and `reasons_lib` module.

Run a script that imports each module and records all non-underscore names:

```python
import importlib
import json
import pkgutil

modules = ["reasons", "reasons_lib"]
for pkg_name in ("reasons", "reasons_lib"):
    pkg = importlib.import_module(pkg_name)
    for mod in pkgutil.iter_modules(pkg.__path__):
        modules.append(f"{pkg_name}.{mod.name}")

surface = {}
for name in sorted(modules):
    mod = importlib.import_module(name)
    surface[name] = sorted(n for n in vars(mod) if not n.startswith("_"))

with open("compat_public_api.json", "w") as f:
    json.dump(surface, f, indent=2)
```

This snapshot is the compatibility contract. It will include intentional names and accidental re-exports such as `dataclass`, `field`, `json`, and `sqlite3`. That is acceptable for the first migration step because the goal is backward compatibility, not immediate API cleanup.

### 2. Replace wildcard shims with explicit imports

For each `reasons_lib/*.py`, replace line 2 with an explicit import list generated from the snapshot.

Example for `reasons_lib/__init__.py`:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import (  # noqa: F401
    Justification,
    Node,
    Nogood,
    # Retained because the old wildcard exposed these accidentally:
    dataclass,
    field,
)

__all__ = [
    "Justification",
    "Node",
    "Nogood",
    "dataclass",
    "field",
]
```

Example shape for `reasons_lib/api.py`:

```python
"""Backwards-compatibility shim — use ``import reasons.api`` instead."""
from reasons.api import (  # noqa: F401
    DEFAULT_DB,
    Justification,
    Network,
    PROPOSALS_SCHEMA,
    Storage,
    build_meta,
    clear_network_cache,
    init_db,
    pubsub,
    # Legacy incidental re-exports from the old wildcard:
    Callable,
    Path,
    combinations,
    copy,
    datetime,
    json,
    logging,
    os,
    re,
    sqlite3,
    sys,
    timezone,
)

__all__ = [ ...same names as above... ]
```

The exact list must be generated from the snapshot, not hand-written from the first 200 lines. `reasons/api.py` has 5,886 lines, so there are almost certainly more public names than the sample shows.

For all 30 listed shim modules plus the three missing wildcard matches, the final shape should be:

```python
from reasons.<module> import (
    name1,
    name2,
    ...
)

__all__ = ["name1", "name2", ...]
```

This satisfies two goals:

- no `import *` remains in `reasons_lib`;
- `from reasons_lib.<module> import *` continues to expose the same names because `__all__` is explicit.

### 3. Decide when to add `__all__` to `reasons` itself

There are two safe options.

**Option A — shims only, initially**

Change only `reasons_lib/*.py`. Leave `reasons` without `__all__` for one release. This is lowest risk because direct consumers of `from reasons.api import *` see no change.

**Option B — add `__all__` to `reasons` after snapshot**

If you add `__all__` to `reasons/__init__.py` or `reasons/api.py`, include every currently public name from the snapshot, including accidental ones, for at least one compatibility window.

For example, `reasons/__init__.py` should temporarily export:

```python
__all__ = [
    "Justification",
    "Node",
    "Nogood",
    # Legacy wildcard behavior:
    "dataclass",
    "field",
]
```

Only after a major-version boundary or explicit deprecation should `__all__` be pruned to the intended API.

### 4. Add compatibility tests

Add tests that compare each shim against the snapshot:

```python
import importlib
import json

with open("compat_public_api.json") as f:
    expected = json.load(f)

for module_name, names in expected.items():
    if not module_name.startswith("reasons_lib"):
        continue
    mod = importlib.import_module(module_name)
    for name in names:
        assert hasattr(mod, name), f"{module_name} lost {name}"
```

Also test wildcard imports from shims:

```python
ns = {}
exec("from reasons_lib import *", ns)
assert "Node" in ns
assert "dataclass" in ns  # currently accidental but compatibility-relevant
```

And test identity, not just presence:

```python
import reasons
import reasons_lib

assert reasons_lib.Node is reasons.Node
assert reasons_lib.Justification is reasons.Justification
```

### 5. Enforce with lint/CI

After the shims are explicit:

- Remove `F403` from the `# noqa` comments.
- Keep `F401` only where needed, or rely on `__all__` to mark re-exports as intentional.
- Add a CI check or lint rule that rejects new `import *` matches.
- Re-run `rg -n 'import\s+\*'` and confirm the count is zero or only intentionally allowlisted files.

### 6. Cleanup only in a later release

Treat accidental names like `reasons_lib.dataclass`, `reasons_lib.json`, `reasons_lib.sqlite3`, etc. as legacy compatibility surface. Do not remove them in the same change that removes wildcard imports. If you want to clean them up later, do it in a major version or with a documented deprecation window.

## Backwards compatibility risks

- `reasons_lib/__init__.py:2` currently exposes `dataclass` and `field` because `reasons/__init__.py` has no `__all__`. An explicit import list that omits them would break `from reasons_lib import dataclass`.
- `reasons_lib/api.py:2` currently re-exports imported implementation names such as `json`, `sqlite3`, `logging`, `Path`, `Storage`, and `Network`. These are not necessarily intended API, but they are observable today.
- Adding `__all__` to `reasons` itself can break direct consumers of `from reasons.api import *` unless the initial `__all__` includes the full snapshot.
- The wildcard inventory is incomplete: the observations show 30 matches but report 33 total. Re-scan before generating any explicit import lists.

## Topics to Explore

- [general] `public-api-snapshot` — Generate the exact non-underscore export list for every `reasons` and `reasons_lib` module before changing any wildcard imports.
- [file] `reasons/api.py` — Enumerate all top-level names in this 5,886-line module and separate intended API from accidental imported re-exports.
- [general] `wildcard-import-inventory` — Re-run `rg -n 'import\s+\*'` to find the three wildcard matches missing from the provided 30-entry list.
- [file] `pyproject.toml` — Confirm packaging still includes both `reasons` and `reasons_lib` and that console scripts remain pointed at `reasons.cli:main` and `reasons.mcp_server:main`.
- [general] `compatibility-tests` — Add tests for `from reasons_lib.api import *`, attribute presence, and identity against the corresponding `reasons` names.

## Beliefs

- `reasons-lib-is-wildcard-shim` — The listed `reasons_lib` modules currently re-export their `reasons` counterparts with `from reasons.<module> import *` on line 2.
- `reasons-package-has-no-all` — No `__all__` exists in the observed `reasons` package, so wildcard exports are implicit.
- `reasons-lib-root-reexports-dataclass` — `reasons_lib/__init__.py:2` currently exposes `dataclass` and `field` in addition to `Justification`, `Node`, and `Nogood`.
- `pyproject-packaging-includes-reasons-lib` — `pyproject.toml:31` packages both `reasons` and `reasons_lib`, so the shims ship with the distribution.
- `cli-entrypoints-bypass-shim` — `pyproject.toml:27-28` targets `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib`, so CLI entry points are independent of the shim migration.
