---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Confirm whether `reasons` has submodules; top-level `sys.modules` aliasing does not automatically alias `reasons.foo` to `reasons_lib.foo`.

## Explanation

`reasons` does have submodules. The observation includes `reasons/api.py`, so `reasons.api` is a real submodule of the `reasons` package. However, the package initializer `reasons/__init__.py` (59 lines) only defines the dataclasses `Justification`, `Node`, and `Nogood`; it does not import `api` or any other submodule into the top-level `reasons` namespace.

The `reasons_lib` compatibility shim is not a module alias. `reasons_lib/__init__.py` line 2 does:

```python
from reasons import *
```

That copies public names from `reasons` into `reasons_lib` at import time. It does **not** execute `sys.modules["reasons_lib"] = reasons`, and it does not create aliases such as `sys.modules["reasons_lib.api"] = reasons.api`.

This is supported by the searches:

- `find_sys_modules_aliasing` found **zero** `sys.modules` matches.
- `find_dynamic_import_aliasing` only found `__import__("os")` in `tests/test_pg_dispatch.py` lines 43, 59, and 85, unrelated to package aliasing.
- `check_test_imports` found no `from reasons.` or `import reasons.` matches, so there is no observed test coverage of `reasons` submodule imports.

`reasons/api.py` imports names from `metadata`, `network`, `storage`, and from an empty module name for `Justification`/`pubsub`. If those are relative imports, they suggest additional `reasons.*` submodules, but the observations do not include the full file tree, so only `reasons.api` is directly confirmed.

Practical consequence:

- `from reasons_lib import Justification` should work because `Justification` is re-exported by the star import.
- `import reasons_lib.api` will not automatically resolve to `reasons.api`; Python looks for an actual `reasons_lib.api` module or an explicit `sys.modules` alias, neither of which is observed.
- `import reasons.api` loads `reasons.api` under the `reasons` package, but does not automatically register `reasons_lib.api`.
- Even a parent-level `sys.modules["reasons_lib"] = reasons` alias would not automatically alias child modules such as `reasons.foo` to `reasons_lib.foo`; child aliases would need explicit handling.

## Topics to Explore

- [file] `reasons/api.py` — Check whether the `metadata`, `network`, `storage`, and `pubsub` imports are relative submodules and map the full `reasons` submodule tree.
- [file] `reasons_lib/__init__.py` — Understand exactly which names `from reasons import *` re-exports and whether an explicit `__all__` or submodule alias strategy is needed.
- [general] `sys-modules-submodule-aliasing` — Explore patterns for explicitly aliasing child modules in `sys.modules` if compatibility requires `reasons_lib.foo` to resolve to `reasons.foo`.
- [file] `tests/test_pg_dispatch.py` — Contains the only dynamic import matches; confirm it is unrelated and consider adding coverage for `reasons_lib` submodule imports.

## Beliefs

- `reasons-has-api-submodule` — The observed file `reasons/api.py` confirms `reasons.api` as a submodule of the `reasons` package.
- `reasons-init-does-not-import-submodules` — The 59-line `reasons/__init__.py` defines dataclasses without importing `api` or sibling modules.
- `reasons-lib-uses-star-import-not-alias` — `reasons_lib/__init__.py` line 2 re-exports names via `from reasons import *`, not via `sys.modules` aliasing.
- `no-sys-modules-aliasing-exists` — The search for `sys.modules` found zero matches, so no parent or child module aliasing is implemented.
- `reasons-lib-api-import-needs-explicit-alias-or-file` — Importing `reasons_lib.api` requires a `reasons_lib/api.py` file or an explicit `sys.modules` alias, neither observed.
