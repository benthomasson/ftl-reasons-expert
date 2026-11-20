---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The mechanism (and its limits) by which `reasons_lib.*` mirrors `reasons.*`; useful before doing any rename/refactor to know which paths to keep alive.

## How `reasons_lib.*` mirrors `reasons.*`

`reasons_lib` is **not a dynamic alias package**. It is a set of static, per-module compatibility shims.

The two observed shims are:

- `reasons_lib/__init__.py` line 2: `from reasons import *`
- `reasons_lib/api.py` line 2: `from reasons.api import *`

Both files are explicitly described as backwards-compatibility shims. Packaging also includes both packages in `pyproject.toml` line 31:

```toml
packages = ["reasons", "reasons_lib"]
```

There is no dynamic forwarding mechanism: `grep_alias_mechanisms` found no matches for `sys.modules`, `__getattr__`, `import_module`, `ModuleType`, `globals()`, or `__path__`. So Python’s normal import rules decide what exists under `reasons_lib`.

### What the star imports actually do

`from reasons import *` executes `reasons/__init__.py` and copies its public names into `reasons_lib`.

`reasons/__init__.py` has **no `__all__`**. It imports `dataclass` and `field` at line 3, then defines `Justification`, `Node`, and `Nogood` starting around lines 6–57. Therefore `reasons_lib` currently exposes at least:

- `dataclass`
- `field`
- `Justification`
- `Node`
- `Nogood`

The imported classes are the same objects, e.g. `reasons_lib.Node is reasons.Node` is true. But they are copied as namespace bindings, not proxied.

For `reasons_lib/api.py`, the exact mirrored surface cannot be fully determined from the observations because `reasons/api.py` is not included. The shim will mirror whatever `from reasons.api import *` imports: if `reasons/api.py` defines `__all__`, only those names; otherwise all non-underscore names in that module.

### Limits of the mirroring

- **Per-file, not recursive.** A new `reasons.foo` does not automatically create `reasons_lib.foo`. You must add a physical `reasons_lib/foo.py` wrapper with its own `from reasons.foo import *`.
- **No module identity aliasing.** `reasons_lib.api` is a distinct module object from `reasons.api`. Imported classes/functions are shared references, but rebinding `reasons.api.SomeName` later does not update `reasons_lib.api.SomeName` unless the shim is reloaded.
- **No automatic submodule mirroring.** `reasons_lib/__init__.py` does not expose `reasons_lib.api` just because `reasons.api` exists. The `api` path works only because `reasons_lib/api.py` exists and is imported as a submodule.
- **Star-import filtering matters.** If `reasons/api.py` has `__all__`, the shim only gets those names. If it lacks `__all__`, the shim may unintentionally expose imported helper modules/names. The top-level `reasons_lib` already does this by exposing `dataclass` and `field`.
- **Rename fragility.** Both shim files contain literal `reasons` imports. Renaming/moving `reasons` or `reasons.api` breaks the shims unless the wrappers are updated. Removing top-level `Justification`, `Node`, or `Nogood` from `reasons/__init__.py` also breaks `reasons_lib` top-level compatibility.
- **Packaging dependency.** If external callers must keep using `reasons_lib`, keep it in `pyproject.toml` line 31 and keep `reasons` installed as its source.

### Paths to keep alive before a rename/refactor

Keep these current compatibility paths working:

- `reasons_lib/__init__.py` and the top-level exports from `reasons/__init__.py`: `Justification`, `Node`, `Nogood`, plus currently `dataclass` and `field` due to the star import.
- `reasons_lib/api.py` and the `reasons.api` module/export surface it imports.
- Any other `reasons_lib/*.py` wrappers not shown in the observations; a full directory listing is missing.
- The `reasons_lib` package entry in `pyproject.toml` line 31.

The observations are insufficient to enumerate every mirrored path because only `reasons_lib/__init__.py` and `reasons_lib/api.py` are shown, and `reasons/api.py` is absent. Also, the empty `grep_intentional_wrappers` result conflicts with the visible star imports, so that grep should not be treated as exhaustive.

## Topics to Explore

- [file] `reasons/api.py` — Needed to enumerate exactly which names `reasons_lib/api.py` mirrors and whether `__all__` narrows the star import.
- [file] `reasons_lib/` — A full directory listing is needed to find every wrapper path that must remain alive.
- [general] `star-import-semantics` — Understand how `from module import *` handles `__all__`, submodules, and rebinding; this defines the shim’s limits.
- [general] `packaging-backcompat` — Verify that `reasons_lib` is included in wheels/sdists and whether external callers rely on it.
- [diff] `rename-reasons-package` — Simulate renaming `reasons` or `reasons.api` to see exactly which shim imports break.

## Beliefs

- `reasons-lib-is-a-static-star-import-shim` — `reasons_lib/__init__.py` and `reasons_lib/api.py` mirror via literal `from reasons... import *` statements, not dynamic module aliasing.
- `reasons-lib-top-level-exports-include-justification-node-nogood` — Because `reasons/__init__.py` has no `__all__`, `reasons_lib` top-level star import exposes `dataclass`, `field`, `Justification`, `Node`, and `Nogood`.
- `reasons-lib-does-not-auto-mirror-new-submodules` — Adding `reasons.foo` does not create `reasons_lib.foo`; a physical `reasons_lib/foo.py` wrapper is required.
- `reasons-lib-module-objects-are-distinct-from-reasons` — `reasons_lib.api` is a separate module object; imported names are shared references, but later rebinding in `reasons.api` is not reflected without reload.
- `packaging-includes-reasons-lib` — `pyproject.toml` line 31 includes `reasons_lib` in the setuptools packages list.
