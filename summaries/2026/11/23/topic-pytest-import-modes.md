---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: How `prepend` vs `append` vs `importlib` change `sys.path` mutation and test-module naming, which is the decisive factor for dropping `tests/__init__.py`.

## How `prepend`, `append`, and `importlib` affect `sys.path` and test-module names

The key point: `tests/__init__.py` is not primarily about packaging here — `pyproject.toml:30-31` packages only `reasons` and `reasons_lib`. It is about **pytest’s import mode and resulting module identity**. In the current repo, `pyproject.toml:33-34` sets only `testpaths = ["tests"]`; it does **not** set an import mode or `pythonpath`. So pytest uses its default `prepend` mode.

### Mode comparison

| Mode | `sys.path` mutation | Module name with `tests/__init__.py` | Module name without `tests/__init__.py` |
|---|---|---|---|
| `prepend` default | Prepends the computed basedir to `sys.path` | `tests.test_foo` | `test_foo` |
| `append` | Appends the computed basedir to `sys.path` | `tests.test_foo` | `test_foo` |
| `importlib` | Does not insert test directories into `sys.path` for test import; loads from file path via `importlib` | Usually package marker is irrelevant for test-module naming | Same: package marker is irrelevant |

More concretely:

- **`prepend`**
  - If `tests/__init__.py` exists, pytest treats `tests` as a package. A file like `tests/test_api.py` is imported as `tests.test_api`, and the repo root is prepended to `sys.path`.
  - If `tests/__init__.py` is removed, pytest treats the test file as a top-level module. `tests/test_api.py` becomes `test_api`, and the `tests/` directory is prepended to `sys.path`.
  - This can shadow other top-level modules because the test directory goes to the front of `sys.path`.

- **`append`**
  - Same package detection and naming rules as `prepend`: `tests/__init__.py` still determines whether modules are named `tests.test_api` or `test_api`.
  - The difference is only where the basedir goes: appended to the end of `sys.path` rather than prepended. That reduces shadowing risk but can also make test helpers fail to resolve if another module with the same name appears earlier on `sys.path`.

- **`importlib`**
  - Pytest imports test modules directly from their file paths using `importlib`.
  - It does not rely on inserting the test directory into `sys.path` to make test modules importable.
  - It also does not use `tests/__init__.py` to decide package-qualified test-module names in the same way `prepend`/`append` do.
  - This mode is often used specifically to avoid `sys.path` mutation and same-basename collisions, at the cost that test modules are not imported as ordinary package modules.

### What is decisive for dropping `tests/__init__.py`

The decisive factor is whether anything depends on `tests` being a Python package:

1. relative imports such as `from .helpers import ...`
2. package-qualified imports such as `from tests.helpers import ...` or `import tests`
3. shared helper modules that are imported by package name
4. duplicate test-module basenames that collide when imported as top-level modules under `prepend`/`append`

The observations show none of the import-style dependencies:

- `relative_or_tests_package_imports` has `total_count: 0` — no `from .`, `from tests`, or `import tests` matches.
- `path_mutation_and_import_mode_in_python` has `total_count: 0` — no application code manually mutates `sys.path` or uses `importlib`.
- `pytest_import_notes_in_markdown` and `pytest_import_notes_in_json` also have no matches.
- `tests/__init__.py:1` is empty.

The shared fixture setup does not require `tests` to be a package:

- `tests/conftest.py:9` defines `pytest_configure`, which pytest loads via conftest discovery, not via `import tests.conftest`.
- `tests/conftest.py:18` defines the `pg_api` fixture.
- `tests/conftest.py:23` imports `from reasons.pg import PgApi` absolutely inside the fixture, so it depends on `reasons` being importable, not on `tests` being importable.

So in this codebase, dropping `tests/__init__.py` is safe from the observed import-naming perspective because no code relies on `tests.*` module names or relative imports inside the tests package. Under the current default `prepend` mode, dropping it will change test modules from `tests.test_*` to top-level `test_*` and prepend `tests/` to `sys.path`; that is acceptable only if there are no duplicate test basenames and no test helper module names that would shadow real dependencies.

If you wanted to drop `tests/__init__.py` while also avoiding `sys.path` mutation and basename collisions, the more robust option would be to configure pytest with `importlib` mode, for example via `addopts = "--import-mode=importlib"` in `pyproject.toml`. But the observations do not show that configuration today.

## Topics to Explore

- [file] `tests/` — Need the full test tree to confirm there are no duplicate test-module basenames that would collide under default `prepend` after removing `tests/__init__.py`.
- [file] `.github/workflows/tests.yml` — Check whether CI runs `pytest` from the repo root, uses `python -m pytest`, or sets any import-mode flags that affect `sys.path`.
- [general] `pytest-import-modes` — Compare the installed pytest version’s exact `prepend`, `append`, and `importlib` behavior for conftest and test-module discovery.
- [file] `pyproject.toml` — Decide whether to add `addopts = "--import-mode=importlib"` or leave the default `prepend` behavior.
- [function] `tests/conftest.py:pg_api` — Verify the fixture’s absolute `reasons.pg` import remains resolvable after any import-mode change.

## Beliefs

- `pytest-config-uses-default-prepend` — `pyproject.toml:33-34` configures only `testpaths`, so pytest uses the default `prepend` import mode.
- `tests-init-is-empty` — `tests/__init__.py:1` is empty and exists only as a package marker.
- `tests-do-not-import-tests-package` — No relative or `tests`-package imports were found, so current tests do not rely on `tests.*` module names.
- `conftest-fixture-imports-reasons-absolutely` — `tests/conftest.py:23` imports `reasons.pg.PgApi` absolutely inside `pg_api`, not through the tests package.
- `setuptools-packages-exclude-tests` — `pyproject.toml:30-31` lists only `reasons` and `reasons_lib`, so `tests/__init__.py` is not needed for distribution packaging.
