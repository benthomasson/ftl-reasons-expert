---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Search downstream users, tests, and docs for remaining `reasons_lib` imports before planning removal.

## Search result summary

The goal is to prove that `reasons_lib` is unused before removing it, while distinguishing **internal references** from **external downstream consumers**.

| Check | Observation | Implication |
|---|---|---|
| All repository references | Reported `total_count: 0` | Suggests no tracked references, but conflicts with packaging/shim observations below. |
| Python imports from `reasons_lib` | `total_count: 0` | No `from reasons_lib ...` or `import reasons_lib ...` statements were found. |
| Test references | `total_count: 0` | No test file mentions `reasons_lib`. |
| Markdown docs references | `total_count: 0` | No Markdown documentation mentions `reasons_lib`. |
| GitHub workflow references | `total_count: 0` | CI workflows do not reference `reasons_lib`. |
| Packaging metadata | `pyproject.toml` `[tool.setuptools]` has `packages = ["reasons", "reasons_lib"]` around line 30 in the returned content | `reasons_lib` is still intentionally packaged. Removing the directory alone would leave packaging metadata inconsistent. |
| Package shim | `reasons_lib/__init__.py` line 1: `from reasons import *  # noqa: F401,F403` | `reasons_lib` is a compatibility facade re-exporting the canonical `reasons` package. |
| Test aliasing config | `tests/conftest.py` lines 1-39 do not alias `reasons_lib`; line 22 imports `from reasons.pg import PgApi` | Tests use the canonical `reasons` package, not a `reasons_lib` compatibility path. |

### Interpretation

Inside the repository, the observed Python import search found **no downstream code importing `reasons_lib`**. Tests also appear not to depend on it: `tests/conftest.py` defines the `pg_api` fixture and imports `PgApi` from `reasons.pg` at line 22, not from `reasons_lib`. There are also no Markdown or workflow references.

However, the package is not fully removed yet. Two artifacts still matter:

1. `pyproject.toml` still includes `"reasons_lib"` in the setuptools package list. If removal is planned, this line must be removed as part of the change.
2. `reasons_lib/__init__.py` is the shim itself. It performs a wildcard re-export from `reasons`, so external code doing `from reasons_lib import X` would continue to work today.

There is also an important inconsistency: the aggregate “all repository references” search reports zero matches, yet `pyproject.toml` contains `"reasons_lib"` and the package path `reasons_lib/__init__.py` exists. That means the aggregate search scope should be validated before relying on the zero count. It may have excluded packaging files, file paths, hidden files, or non-source file types.

### Removal planning implications

Before removal, confirm all of the following:

- Re-run the search across all file types and paths, including `pyproject.toml`, package paths, docs formats beyond Markdown, notebooks, scripts, and generated files.
- Search for dynamic imports such as `importlib.import_module("reasons_lib")`, `__import__("reasons_lib")`, and string-based `sys.modules` manipulation.
- Decide whether `reasons_lib` is a published compatibility API. Repository search cannot prove external downstream packages do not import it.
- If removal is safe, remove both the shim file and the `pyproject.toml` package entry, and add a changelog/deprecation note if the alias was public.
- If external use is possible, prefer a deprecation cycle: keep the shim, emit a `DeprecationWarning`, then remove in a future major release.

## Topics to Explore

- [file] `pyproject.toml` — Verify the `[tool.setuptools]` package list and plan removal of `"reasons_lib"` from packaging metadata.
- [file] `reasons_lib/__init__.py` — Inspect the compatibility shim and decide whether to delete it immediately or deprecate it first.
- [function] `tests/conftest.py:pg_api` — Confirms tests import from canonical `reasons.pg`, not `reasons_lib`, so tests likely do not require the shim.
- [general] `dynamic-import-search` — Search for `importlib`, `__import__`, and string references that static import-pattern searches miss.
- [general] `external-downstream-users` — Check release history, package consumers, and support issues to determine whether external users may still import `reasons_lib`.

## Beliefs

- `no-python-imports-reasons-lib` — The observed Python import search found zero `from reasons_lib` or `import reasons_lib` statements.
- `pyproject-packages-contains-reasons-lib` — `pyproject.toml` still includes `"reasons_lib"` in the setuptools package list, so removal requires a packaging metadata change.
- `reasons-lib-re-exports-reasons` — `reasons_lib/__init__.py` line 1 uses `from reasons import *`, making `reasons_lib` a compatibility facade over `reasons`.
- `tests-conftest-does-not-alias-reasons-lib` — `tests/conftest.py` contains no `reasons_lib` import aliasing and imports `PgApi` from `reasons.pg` at line 22.
- `repo-search-zero-conflicts-with-observed-metadata` — The aggregate search reports zero `reasons_lib` references even though `pyproject.toml` and the `reasons_lib` package path contain it, so the search scope must be validated.
