---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Checks whether any test imports through `reasons_lib` to catch stale shim behavior.

# No check currently exists to enforce that tests avoid `reasons_lib`

The intent described — "checks whether any test imports through `reasons_lib` to catch stale shim behavior" — is **not implemented** in the observed codebase. Every search that would reveal such a guard returns zero matches.

## What the observations show

### The shim itself exists

`reasons_lib/__init__.py` (3 lines) is a pure re-export shim:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

It is registered as a shipped package in `pyproject.toml` under `[tool.setuptools]` (`packages = ["reasons", "reasons_lib"]`), so it is still installed for consumers relying on the old import path.

### Nothing references it

Three independent greps for the literal string `reasons_lib` all return `total_count: 0`:

| Search scope | Pattern | Matches |
|---|---|---|
| Tests | `reasons_lib` | 0 |
| All Python files | `reasons_lib` | 0 |
| GitHub workflows | `reasons_lib` | 0 |

Notably, even the shim's own source doesn't literally contain `reasons_lib` (its docstring says "Backwards-compatibility shim" and it imports `from reasons import *`), so a raw grep for the string only succeeds against `pyproject.toml`, which was read but not included in the "find" scopes.

### The supporting infrastructure has no guard

- `tests/conftest.py` (40 lines) defines only `pytest_configure`, `pg_available`, `skip_no_pg`, and the `pg_api` fixture. There is **no hook, meta-test, or collection-level assertion** that inspects `sys.modules` or scans test sources for `reasons_lib`.
- The only CI file is `.github/workflows/tests.yml`; the workflow search confirms it contains no `reasons_lib` reference, so CI does not fail-fast on shim usage either.
- No test file imports the shim (the tests-scoped grep is empty), so the shim is currently **unused by the test suite** — but that fact is incidental, not enforced.

## Practical interpretation

If someone claims a check exists that "catches stale shim behavior," the observations contradict that claim. The current state is:

1. **Desired property**: no test should import from `reasons_lib` (so the deprecated shim can eventually be deleted).
2. **Actual state**: no test imports it (grep = 0), but **nothing would fail if one started to** — there is no assertion, no `conftest.py` hook, and no CI lint step that would flag a new `import reasons_lib` in `tests/`.

The natural place to add such a guard would be `tests/conftest.py:pytest_configure` (a `pytest_collection_modifyitems` hook) or a tiny meta-test that walks `tests/**/*.py` and asserts `"reasons_lib"` doesn't appear — plus a step in `.github/workflows/tests.yml` to fail CI on a positive match. None of those are shown in the observations.

## What's missing

The observation set does **not** include the content of `.github/workflows/tests.yml` or of any individual test module, only aggregate grep counts. To fully confirm that no test (e.g., via a string-based check or imported helper) imports the shim, those files would need to be inspected. However, the aggregate counts are strong evidence that the guard does not exist today.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Confirm no lint/step that greps for `reasons_lib` usage in tests; this is where CI enforcement would live
- [file] `reasons_lib/__init__.py` — The shim that would become dead code once all consumers migrate to `reasons`
- [function] `tests/conftest.py:pytest_configure` — The natural hook point for a `pytest_collection_modifyitems` assertion that would actually implement the described check
- [file] `pyproject.toml` — The `packages = ["reasons", "reasons_lib"]` entry that keeps the shim installed and must be removed to fully retire it
- [general] `shim-retirement-policy` — Whether `reasons_lib` is scheduled for removal in a future version, which would make a guard-then-delete sequence concrete

## Beliefs

- `no-test-imports-reasons_lib` — A grep for `reasons_lib` across `tests/` returns zero matches.
- `reasons_lib-is-a-shim` — `reasons_lib/__init__.py` re-exports everything from `reasons` via `from reasons import *`.
- `reasons_lib-listed-in-pyproject` — `pyproject.toml` ships both `reasons` and `reasons_lib` as installed packages.
- `conftest-has-no-shim-guard` — `tests/conftest.py` contains no reference to `reasons_lib` and no import-inspection hook.
- `ci-has-no-shim-guard` — The only workflow (`.github/workflows/tests.yml`) contains no `reasons_lib` reference, so shim usage in tests is not currently caught by CI.
