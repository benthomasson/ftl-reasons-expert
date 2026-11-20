---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The repository mirrors modules under both packages; confirm which package CI tests and whether both need changes.

## Explanation

**Short answer:** The observations do **not** confirm that CI tests either package. The only workflow is `.github/workflows/tests.yml`, but the search for `pytest|python -m|reasons_lib|reasons` in CI workflows returned **0 matches**. So we cannot confirm that CI runs pytest at all, let alone which package it exercises. The actual contents of `.github/workflows/tests.yml` are needed.

What the repository config does show:

- `pyproject.toml` line 30 declares both packages in the distribution:  
  `packages = ["reasons", "reasons_lib"]`.
- `pyproject.toml` line 33 points pytest at `tests/`:  
  `testpaths = ["tests"]`.
- `pyproject.toml` lines 26–27 define CLI entry points against `reasons.*`:  
  `reasons = "reasons.cli:main"` and `reasons-mcp = "reasons.mcp_server:main"`. That makes `reasons` the canonical package.
- `tests/conftest.py`’s `pg_api` fixture imports from `reasons`:  
  `from reasons.pg import PgApi` (around line 22). The visible test fixture is therefore written against `reasons`, not `reasons_lib`.
- `CLAUDE.md:24` says `reasons/` is importable as `import reasons` and is also aliased as `reasons_lib` for backwards compatibility.
- `CLAUDE.md:35` says `reasons_lib/` is a **backwards-compat shim** that re-exports from `reasons/`.
- The earlier search `find_which_package_tests_import` returned 0, but its regex is anchored with `^`, and `conftest.py`’s import is indented inside a fixture. That zero is a false negative; the file content is stronger evidence.
- There is no observed package-sync automation: search for `reasons_lib|rsync|cp -r|mirror|sync` in CI/sync scripts returned 0. So if `reasons_lib` is a duplicated mirror rather than a shim, nothing observed keeps it in sync.

**So, which package needs changes?**

- Implementation changes should go in `reasons/` first, because tests and CLI entry points reference it.
- `reasons_lib/` needs changes only if the public compatibility surface changes and the shim must re-export new or changed symbols.
- If `reasons_lib/` is actually duplicated mirrored code rather than a shim, then both packages need changes. But the observations do **not** prove duplication; they document `reasons_lib` as a shim.
- CI is not shown to test `reasons_lib` directly. If backwards compatibility matters, add or verify a test that imports through `reasons_lib`; otherwise a stale shim can ship unnoticed.

Missing information: the actual contents of `.github/workflows/tests.yml`, the contents of `reasons_lib/`, and the rest of `tests/` to see whether any test imports `reasons_lib`.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Confirms the actual CI command and whether CI runs pytest directly, through a wrapper, or via a reusable workflow.
- [general] `reasons-lib-shim-vs-mirror` — Determines whether `reasons_lib/` re-exports from `reasons/` or contains duplicated mirrored modules.
- [file] `tests/conftest.py` — Verifies the `pg_api` fixture and whether other test fixtures or tests import `reasons_lib`.
- [file] `pyproject.toml` — Confirms package inclusion, pytest testpaths, and optional test dependencies.
- [general] `backwards-compat-test-coverage` — Checks whether any test imports through `reasons_lib` to catch stale shim behavior.

## Beliefs

- `tests-import-reasons` — `tests/conftest.py` imports `PgApi` from `reasons.pg` inside the `pg_api` fixture.
- `reasons-lib-re-exports-reasons` — `CLAUDE.md:35` describes `reasons_lib/` as a backwards-compat shim that re-exports from `reasons/`.
- `pytest-collects-tests-directory` — `pyproject.toml` sets `[tool.pytest.ini_options] testpaths = ["tests"]`.
- `ci-test-command-unconfirmed` — The only workflow search found no `pytest`, `python -m`, `reasons_lib`, or `reasons` matches in `.github/workflows`.
- `no-observed-package-sync-automation` — Search for `rsync`, `cp -r`, `mirror`, or `sync` in CI/sync scripts returned zero matches.
