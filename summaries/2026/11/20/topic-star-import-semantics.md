---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Python `__all__`, wildcard import, and re-export behavior are central to this file.

## Explanation

### Core mechanics

Python’s `__all__` controls what `from module import *` imports:

- If a module defines `__all__ = ["Name1", "Name2"]`, wildcard import imports only those names.
- If `__all__` is absent, wildcard import imports every name in the module’s global namespace that does **not** start with `_`.
- That means wildcard imports include names that were themselves imported into the module. Python does not distinguish “defined here” from “imported here” unless `__all__` says so.
- `__all__` does **not** affect explicit imports like `from reasons import dataclass` or attribute access like `reasons.dataclass`. It only controls wildcard export and related tooling/documentation behavior.

Re-exporting via wildcard is therefore a namespace copy, not a live proxy. If `module_a` does `from module_b import *`, the current bindings are copied into `module_a`. Whether a later `from module_a import *` exposes them depends on `module_a.__all__`, or on the implicit “all non-underscore names” rule if `__all__` is missing.

### What happens in this repo

`reasons/__init__.py` is the core data-model package file. It imports `dataclass` and `field` from `dataclasses` at line 3, then defines `Justification`, `Node`, and `Nogood`. It does **not** define `__all__`. The repository-wide grep confirms this: `grep_all_dunder.total_count == 0`.

So, for `reasons/__init__.py`, this:

```python
from reasons import *
```

effectively imports the non-underscore names currently bound in `reasons`, including:

- `Justification`
- `Node`
- `Nogood`
- `dataclass`
- `field`

That last pair is usually not intended public API, but it is exposed because there is no `__all__`.

Now look at `reasons_lib/__init__.py:2`:

```python
from reasons import *  # noqa: F401,F403
```

Because `reasons` has no `__all__`, `reasons_lib/__init__.py` receives `Justification`, `Node`, `Nogood`, `dataclass`, and `field`. Since `reasons_lib/__init__.py` also has no `__all__`, a downstream `from reasons_lib import *` re-exports those same public names. The `# noqa: F401,F403` only silences linter warnings; it does not constrain the runtime namespace.

This is the compatibility-shim behavior: `reasons_lib` is intended as a backwards-compatible alias layer, but the wildcard re-export is broader than the intended model classes. Adding `__all__ = ["Justification", "Node", "Nogood"]` to `reasons/__init__.py` would prevent `dataclass` and `field` from entering `reasons_lib` through line 2. Adding `__all__` only to `reasons_lib/__init__.py` would control `from reasons_lib import *`, but it would not remove already-bound attributes like `reasons_lib.dataclass`.

The same pattern repeats across the package. The observation reports 33 wildcard imports, including `reasons_lib/api.py:2`, `reasons_lib/ask.py:2`, and many other shim modules:

```python
from reasons.api import *  # noqa: F401,F403
from reasons.ask import *  # noqa: F401,F403
```

Because no module in the repo defines `__all__`, each of these shims re-exports all non-underscore names from its target module. That includes imported names, not just names defined in that target module. For example, `reasons/api.py:22-23` imports `Justification` and `pubsub` at module scope:

```python
from . import Justification
from . import pubsub
```

So `from reasons_lib.api import *` can expose `Justification` and `pubsub` in addition to the public API of `reasons.api`. The observations do not show the full `reasons/api.py` context, so lines like `reasons/api.py:2242` and `reasons/api.py:2555` may be function-local imports and may not affect the module namespace; that detail is missing. But the top-level imports at lines 22–23 do affect wildcard export.

Finally, `pyproject.toml` packages both `reasons` and `reasons_lib`, so this compatibility behavior ships as public distribution surface. Any future module-level import added to `reasons/__init__.py` automatically becomes part of `reasons_lib`’s wildcard API unless `__all__` is added.

## Topics to Explore

- [file] `reasons/api.py` — Inspect which `from . import ...` statements are module-level versus function-local to determine exactly which names leak through `reasons_lib/api.py:2`.
- [general] `implicit-public-api-audit` — Compare `dir(reasons)`, `dir(reasons_lib)`, and each `reasons_lib/*.py` shim against the intended documented API.
- [general] `explicit-reexport-migration` — Plan the migration from wildcard shims to explicit imports and/or `__all__` while preserving backwards compatibility.
- [file] `pyproject.toml` — Confirm how `reasons_lib` is packaged and whether any entry points or consumers depend on the shim namespace.
- [function] `reasons/cli.py:main` — Check whether CLI code relies on package-level re-exports or imports submodules explicitly.

## Beliefs

- `reasons-init-has-no-all` — `reasons/__init__.py` defines no `__all__`, so `from reasons import *` uses Python’s implicit public-name export rule.
- `reasons-lib-init-reexports-dataclass-and-field` — `reasons_lib/__init__.py:2` re-exports `dataclass` and `field` because `reasons/__init__.py` imports them at module scope and has no `__all__`.
- `reasons-lib-init-has-no-all` — `reasons_lib/__init__.py` defines no `__all__`, so its own wildcard export is not constrained to the intended compatibility API.
- `wildcard-shims-are-unfiltered` — The repo has 33 wildcard imports and zero `__all__` declarations, so each `reasons_lib/*.py` wildcard shim re-exports all non-underscore names from its target module.
- `reasons-lib-is-packaged-shim` — `pyproject.toml` packages both `reasons` and `reasons_lib`, so the wildcard compatibility behavior is part of the distributed package surface.
