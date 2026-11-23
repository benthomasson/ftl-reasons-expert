---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Check consumers that rely on `reasons_lib.__name__`, `__spec__`, reload behavior, pickling, or submodule imports.

## Findings

The static searches show **no in-repo consumers** for any of the targeted behaviors:

- `reasons_lib.__name__`: 0 matches
- `reasons_lib.__spec__`: 0 matches
- `importlib.reload` / `reload(`: 0 matches
- `pickle`: 0 matches
- `import reasons_lib` / `from reasons_lib ...`: 0 matches

The only relevant artifact is `reasons_lib/__init__.py`. It is a compatibility shim whose docstring says to use `import reasons` instead, and whose only substantive line is `from reasons import *` at line 1 in the observed 0-indexed content.

`reasons/__init__.py` defines the public dataclasses `Justification`, `Node`, and `Nogood`, and it does **not** define `__all__`. That means the star import into `reasons_lib` re-exports all public names from `reasons`, including the dataclasses and also imported names like `dataclass` and `field`.

### Concern-by-concern

| Concern | Evidence | Implication |
|---|---|---|
| `reasons_lib.__name__` | `find_reasons_lib___name___consumers` has 0 matches | No in-repo code directly reads `reasons_lib.__name__`. Python still sets it to `"reasons_lib"`; the star import does not change the shim’s module name. |
| `reasons_lib.__spec__` | `find_reasons_lib___spec___consumers` has 0 matches | No in-repo code inspects the shim’s module spec. The import system still sets `__spec__`, which is what `importlib.reload` relies on. |
| Reload behavior | `find_reload_usage` has 0 matches | No in-repo reload usage. If `reasons_lib` is reloaded, only the shim is re-executed. It does not reload `reasons`; it just re-runs `from reasons import *`. Reloading `reasons` alone would not refresh `reasons_lib`’s already-bound names until `reasons_lib` is also reloaded. |
| Pickling | `find_pickle_usage` has 0 matches | No in-repo pickle usage. The dataclasses are defined in `reasons/__init__.py`, so new pickles of `reasons_lib.Node`, for example, will normally reference `reasons.Node`, not `reasons_lib.Node`, because pickle uses the class’s `__module__`. Old pickles referencing `reasons_lib.Node` could still unpickle through the shim if the name is exported. |
| Submodule imports | `find_reasons_lib_submodule_imports` has 0 matches | No in-repo code imports `reasons_lib` or any `reasons_lib.*` submodule. The pattern would also catch plain `import reasons_lib`, so this is a broad zero. `pyproject.toml` lists only `packages = ["reasons", "reasons_lib"]`, with no `reasons_lib.*` subpackages configured. |

### Caveat

These observations are enough to conclude there are no **static, in-repo** consumers. They do not prove there are no external consumers, dynamic imports via `getattr`, `importlib.import_module`, plugin discovery, or historical pickle files outside this repository. Those would need runtime or external-package inspection.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Only compatibility shim; verify whether it should define `__all__` or explicit re-exports.
- [file] `reasons/__init__.py` — Public dataclass definitions and star-import surface; check `__module__` values and pickle compatibility.
- [file] `pyproject.toml` — Confirm packaging only includes `reasons` and `reasons_lib` and no `reasons_lib.*` subpackages.
- [general] `external-consumers` — Search installed packages, plugins, and historical pickles outside this repo for `reasons_lib` references.
- [general] `dynamic-import-search` — Static grep misses `getattr(module, "__name__")`, string-built imports, and `importlib.import_module`.

## Beliefs

- `reasons-lib-is-backwards-compatibility-shim` — `reasons_lib/__init__.py` only re-exports `reasons` public names via `from reasons import *`.
- `reasons-lib-has-no-in-repo-import-consumers` — Static search found zero `import reasons_lib` or `from reasons_lib ...` matches in the repo.
- `reasons-dataclasses-pickle-via-reasons-module` — `Justification`, `Node`, and `Nogood` are defined in `reasons/__init__.py`, so new pickles reference `reasons.*`, not `reasons_lib.*`.
- `reasons-lib-reload-rebinds-star-imports` — Reloading `reasons_lib` re-executes `from reasons import *`, but does not clear stale names from the shim namespace.
- `reasons-lib-package-has-no-listed-subpackages` — `pyproject.toml` lists only `reasons` and `reasons_lib`; no `reasons_lib.*` subpackage is configured.
