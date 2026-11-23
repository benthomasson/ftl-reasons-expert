---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Whether tests verify `import reasons_lib`, `from reasons_lib.api import ...`, and other historical import paths.

## Whether tests verify historical import paths

**No — based on the observations, the test suite does not verify `import reasons_lib`, `from reasons_lib.api import ...`, or other historical `reasons_lib.*` import paths.**

Evidence:

- The search for `reasons_lib` in tests returned **0 matches**:
  - `test_references_to_reasons_lib.total_count == 0`
  - `test_references_to_reasons_lib.matches == []`

- `tests/test_api.py` imports the **current** package, not the historical shim:
  - `tests/test_api.py:5` → `from reasons import api`
  - The file then tests `api.init_db`, `api.add_node`, etc.
  - `TestPropagate.test_with_changes` also imports `from reasons.storage import Storage`.
  - No `reasons_lib` imports appear in the provided test content.

- The historical compatibility layer exists but is not exercised by tests:
  - `reasons_lib/__init__.py:1` documents itself as a “Backwards-compatibility shim — use `import reasons` instead.”
  - `reasons_lib/__init__.py:3` does `from reasons import *`.
  - `reasons_lib/api.py:1` documents itself as a shim — “use `from reasons.api` instead.”
  - Many other `reasons_lib/*.py` files are also shims, e.g. `reasons_lib/ask.py:1`, `reasons_lib/cli.py:1`, `reasons_lib/contradictions.py:1`.

- The package configuration still ships `reasons_lib`:
  - `pyproject.toml` `[tool.setuptools]` includes `packages = ["reasons", "reasons_lib"]`.

One caveat: the aggregate search result `test_import_statements_for_both_packages.total_count == 0` is internally inconsistent with `tests/test_api.py:5`, which clearly contains `from reasons import api`. That suggests the search extraction may be unreliable for that pattern. It does **not** change the main finding: there are no detected test references to `reasons_lib`.

So: current `reasons` imports are tested indirectly through `tests/test_api.py`; historical `reasons_lib` imports are not verified.

## Topics to Explore

- [file] `tests/test_api.py` — Confirms the current API tests import `from reasons import api`, not the historical `reasons_lib` shim.
- [file] `reasons_lib/api.py` — Shows the compatibility shim implementation that should be smoke-tested for re-export parity.
- [file] `pyproject.toml` — Verifies that `reasons_lib` is still packaged and therefore still part of the installable surface area.
- [general] `backwards-compatibility-import-tests` — Worth designing tests that import every `reasons_lib.*` shim and assert key symbols are identical to `reasons.*`.
- [file] `reasons_lib/__init__.py` — The top-level shim delegates via `from reasons import *`; worth checking whether this exposes the expected public API.

## Beliefs

- `tests-do-not-import-reasons-lib` — The test search for `reasons_lib` returns zero matches.
- `test-api-imports-current-package` — `tests/test_api.py:5` imports `from reasons import api`, exercising the current package rather than `reasons_lib.api`.
- `reasons-lib-is-packaged` — `pyproject.toml` includes both `reasons` and `reasons_lib` in `[tool.setuptools] packages`.
- `reasons-lib-shims-are-backwards-compatibility` — `reasons_lib/*.py` files are explicitly documented as backwards-compatibility shims for `reasons.*`.
- `historical-import-paths-are-untested` — The observed tests do not exercise `import reasons_lib` or `from reasons_lib.* import ...` paths.
