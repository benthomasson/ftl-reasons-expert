---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Decide between explicit imports, `__all__` mirroring, and `sys.modules` aliasing; each has different implications for type checkers, Sphinx autosummary, and pickle compatibility.

## Recommendation

Use **explicit re-exports plus mirrored `__all__`** as the default. Treat `sys.modules` aliasing as a targeted compatibility tool, not the primary shim mechanism.

The current code uses naked star imports in both shims:

- `reasons_lib/__init__.py:2` — `from reasons import *`
- `reasons_lib/api.py:2` — `from reasons.api import *`

There is no `__all__` anywhere and no `sys.modules` aliasing in the observations. `reasons/__init__.py` defines the public dataclasses at `reasons/__init__.py:7` (`Justification`), `:27` (`Node`), and `:53` (`Nogood`), but it also imports `dataclass` and `field` at `reasons/__init__.py:3` without an `__all__`. That means the star import in `reasons_lib/__init__.py:2` leaks `dataclass` and `field` into the compatibility namespace too.

## Option comparison

| Strategy | Type checkers | Sphinx autosummary | Pickle compatibility |
|---|---|---|---|
| **Explicit imports** | Best. Mypy/pyright can see `reasons_lib.Node is reasons.Node`-style aliases directly. | Best. Re-exported names are real module attributes, so autosummary/autodoc can find and list them predictably. | Good for old paths if you explicitly re-export every historical name. Old pickles referencing `reasons_lib.Node` resolve because `reasons_lib.Node` exists. |
| **`__all__` mirroring** | Good when the canonical module defines `__all__` and the shim mirrors it. It bounds star exports and avoids leaking imports like `dataclass`/`field`. | Helpful. `automodule`/autosummary can use `__all__` to decide what belongs in the public surface. Reduces accidental docs for imported helpers. | Same as explicit imports for names listed in `__all__`, but `__all__` does not create submodule aliases by itself. |
| **`sys.modules` aliasing** | Usually opaque to static checkers. Dynamic `sys.modules[...] = ...` is not reliably understood without stubs or explicit imports. | Can confuse autosummary because `reasons_lib` becomes the same module object as `reasons`, or because Sphinx sees duplicate canonical paths. | Strong for old module paths if aliases are installed for every historical module, including submodules. New pickles still use the class’s real `__module__`, which is `reasons`. |

## Important pickle detail

Pickle records `obj.__class__.__module__` and `__qualname__`. The classes here are defined in `reasons/__init__.py`, so their `__module__` is `reasons`, not `reasons_lib`.

That means:

- Old pickles that contain `reasons_lib.Node` can still load if `import reasons_lib; reasons_lib.Node` resolves to the correct class object. The current star shim does that imperfectly; explicit imports do it deliberately.
- New pickles created from these classes will record `reasons.Node`, not `reasons_lib.Node`.
- If old pickles reference submodule paths such as `reasons_lib.api.SomeClass`, `reasons_lib/api.py` must re-export `SomeClass`, or you must install a `sys.modules["reasons_lib.api"]` alias.

The observations contain no pickle-related code (`find_pickle_related_code` returned no matches), so the actual historical pickle surface is not visible here.

## Practical shim shape

For `reasons_lib/__init__.py`, prefer:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import Justification, Node, Nogood

__all__ = ["Justification", "Node", "Nogood"]
```

And add a matching `__all__` in `reasons/__init__.py`:

```python
__all__ = ["Justification", "Node", "Nogood"]
```

For `reasons_lib/api.py`, the exact names depend on `reasons/api.py`, which is not included in the observations. Mirror `reasons.api.__all__` or use explicit imports once that surface is known.

Use `sys.modules` aliasing only if you need runtime module identity, for example if old code does `reasons_lib.api is reasons.api`, or if old pickle paths exist that are not covered by the shim files. If you do use aliasing, add it in addition to explicit re-exports so type checkers and Sphinx still have a static surface to inspect.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the public API and whether it already defines `__all__`; `reasons_lib/api.py` must mirror that surface.
- [file] `reasons/__init__.py` — Add a bounded `__all__` so star imports stop leaking `dataclass` and `field` from line 3.
- [general] `pickle-compatibility-audit` — Search for old pickles, fixtures, or tests that load `reasons_lib.*` qualified names.
- [general] `sphinx-autosummary-config` — Check whether autosummary documents `reasons_lib` separately from `reasons` and whether duplicate-object warnings occur.
- [file] `pyproject.toml` — Confirm both packages are installed via line 31 and check for missing type-checker or docs configuration.

## Beliefs

- `reasons-lib-init-uses-star-import` — `reasons_lib/__init__.py` re-exports the canonical package via `from reasons import *` on line 2.
- `reasons-lib-api-uses-star-import` — `reasons_lib/api.py` re-exports `reasons.api` via `from reasons.api import *` on line 2.
- `reasons-init-lacks-all` — `reasons/__init__.py` defines no `__all__`, so its star export includes `dataclass` and `field` imported at line 3.
- `no-sys-modules-aliasing` — The codebase contains no `sys.modules` aliasing matches according to the observations.
- `dataclasses-are-defined-in-reasons` — `Justification`, `Node`, and `Nogood` are defined in `reasons/__init__.py`, so their `__module__` is `reasons`, not `reasons_lib`.
