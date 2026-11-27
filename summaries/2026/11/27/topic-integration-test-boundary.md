---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Determine whether CI has live-service tests that require secrets versus unit tests that only use monkeypatched env vars.

## Findings

The observations are **not sufficient to prove exactly what CI runs**, because the actual contents of `.github/workflows/tests.yml` were not returned. The only CI evidence is a search over the workflow that returned **zero** matches for `secrets.`, `env:`, `pytest`, `unittest`, `tox`, `nox`, `API_KEY`, `TOKEN`, `DATABASE_URL`, etc. That means there is **no observed CI secret wiring and no observed test command** in the workflow.

However, the test suite itself does contain an optional live-service path:

- `tests/conftest.py:8-9` registers a `pg` marker described as `"requires PostgreSQL (set DATABASE_URL)"`.
- `tests/conftest.py:12` sets `pg_available = bool(os.environ.get("DATABASE_URL"))`.
- `tests/conftest.py:13` defines `skip_no_pg` using `pytest.mark.skipif(...)`.
- `tests/conftest.py:17-29` defines the `pg_api` fixture. It skips at lines 19-20 when `DATABASE_URL` is absent. If present, it imports `PgApi` at line 22, reads the real environment value at line 24, and creates a real `PgApi(conninfo, domain_id)` connection at line 26 before calling `api.init_db()` at line 27. Cleanup deletes test rows from real Postgres tables at lines 32-37.

So the codebase supports **live PostgreSQL integration tests**, but those tests are intentionally gated by `DATABASE_URL`. If CI does not set `DATABASE_URL`, tests using the `pg_api` fixture will skip rather than fail.

The evidence for monkeypatched environment variables is weaker and internally inconsistent: `search_tests_for_env_monkeypatching` returned zero matches, yet `tests/conftest.py` uses `os.environ.get` at line 12 and `os.environ["DATABASE_URL"]` at line 24. That search result is therefore either scoped incorrectly or incomplete. There is no shown evidence of `monkeypatch.setenv`-based unit tests, and no evidence that all unit tests rely only on monkeypatched env vars.

`pyproject.toml` reinforces optional Postgres support: the `test` extra includes `pytest`, `pytest-cov`, and `pyyaml` at line 14, while `pg` and `test-pg` add `psycopg[binary]` at lines 15-16. Pytest is configured with `testpaths = ["tests"]` at line 33.

**Bottom line:** Based on these observations, CI is not shown to require secrets. The repo has optional live-service PostgreSQL tests that require `DATABASE_URL` and skip when it is absent. There is no confirmed CI secret injection, and no confirmed monkeypatched-env-only unit test pattern. To determine this definitively, the actual `.github/workflows/tests.yml` steps and the test files under `tests/` are still needed.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — Read the actual CI steps to see whether it sets `DATABASE_URL`, injects secrets, or runs pytest directly.
- [file] `tests/conftest.py` — Confirm how the `pg_api` fixture and `pg` marker are used by test files.
- [general] `test-search-scope` — The env/secret search returned zero matches despite `tests/conftest.py` using `os.environ`; determine whether `conftest.py` was excluded or the search result is incomplete.
- [file] `pyproject.toml` — Verify optional dependency groups and pytest configuration for expected test modes.
- [general] `database-url-secret-handling` — Determine whether `DATABASE_URL` is supplied as a CI secret, a local opt-in variable, or both.

## Beliefs

- `ci-workflow-has-no-observed-secrets-reference` — The only CI workflow search found no `secrets.` or `env:` references in `.github/workflows/tests.yml`.
- `conftest-database-url-enables-pg-tests` — `tests/conftest.py` makes PostgreSQL integration tests available only when `DATABASE_URL` is set.
- `conftest-postgres-tests-skip-without-database-url` — The `pg_api` fixture skips when `DATABASE_URL` is absent, so CI without that env var will not run live Postgres tests through that fixture.
- `pg-tests-use-live-pgapi-connection` — `pg_api` instantiates `PgApi` with the real `DATABASE_URL` and calls `init_db`, so it is an integration fixture rather than a mocked unit fixture.
- `monkeypatch-search-evidence-conflicts-with-conftest` — The zero-match environment-variable search conflicts with `os.environ` usage in `tests/conftest.py`, so the search scope is incomplete or unreliable.
