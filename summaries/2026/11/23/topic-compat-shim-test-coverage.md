---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Look for pytest cases that import every name through `reasons_lib` to detect wildcard-re-export regressions.

## Findings

The requested pytest coverage is **not present** in the observed codebase.

- `reasons_lib/__init__.py` is a thin compatibility shim:
  - Line 1: docstring: `"""Backwards-compatibility shim — use import reasons instead."""`
  - Line 2: `from reasons import *  # noqa: F401,F403`
- Searches found:
  - `reasons_lib` in tests: **0 matches**
  - `from reasons_lib import`: **0 matches**
  - dynamic import patterns such as `import *`, `getattr(...)`, or `dir(...)` in tests: **0 matches**
  - `__all__` definitions: **0 matches**
- The only matches for “re-export”/“export” are in `tests/test_metadata_extended.py` at lines 4, 64, and 113, but those refer to JSON metadata export/import round-trips, not Python wildcard re-exports.
- `tests/test_api.py:7` imports directly from `reasons`:
  ```python
  from reasons import api
  ```
  It does not exercise `reasons_lib`.

## Conclusion

There are **no pytest cases that import every name through `reasons_lib`**, so there is no existing regression test for the wildcard re-export behavior of `reasons_lib`.

The shim currently relies entirely on:

```python
from reasons import *
```

Because the observations also show no `__all__` definitions anywhere, the shim has no explicit export list of its own. If `reasons` changes its public names, or if the wildcard import behavior changes, no observed test would detect that `reasons_lib` no longer exposes the expected names.

A missing test would likely need to compare public names from `reasons` against names available through `reasons_lib`, for example by iterating over `dir(reasons)` and asserting `hasattr(reasons_lib, name)` for public names. To define the exact expected contract, the contents of `reasons/__init__.py` would also need to be inspected; that file is not included in the observations.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Contains the wildcard re-export shim and has no explicit `__all__`.
- [file] `reasons/__init__.py` — Defines the public names that `from reasons import *` re-exports; needed to design a precise compatibility test.
- [general] `wildcard-reexport-compat-test` — A pytest comparing public names in `reasons` vs. `reasons_lib` would detect wildcard re-export regressions.
- [file] `tests/test_api.py` — Existing tests import directly from `reasons`, not through the backwards-compatibility shim.
- [file] `tests/test_metadata_extended.py` — The “re-export” wording here is about metadata JSON round-trips, not Python package re-exports.

## Beliefs

- `reasons-lib-implements-wildcard-reexport` — `reasons_lib/__init__.py` line 2 re-exports names via `from reasons import *`.
- `reasons-lib-lacks-explicit-all` — The observed codebase contains zero `__all__` definitions, so the shim has no explicit export list.
- `tests-do-not-reference-reasons-lib` — Searches found zero `reasons_lib` references and zero `from reasons_lib import` statements in tests.
- `test-api-imports-reasons-directly` — `tests/test_api.py:7` imports `api` from `reasons`, not from `reasons_lib`.
- `wildcard-reexport-regression-detection-absent` — No observed pytest case compares `reasons` exports with `reasons_lib` exports.
