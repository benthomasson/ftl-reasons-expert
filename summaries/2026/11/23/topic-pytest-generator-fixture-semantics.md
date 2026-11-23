---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Explore pytest’s finalization rules for setup exceptions, skip exceptions, and scoped fixtures.

Line numbers below are inferred from the content shown.

## Observed fixture and pytest config

`tests/conftest.py` contains the only concrete fixture definition in the observations: `pg_api` at `tests/conftest.py:17-39`. It is a generator fixture:

- `tests/conftest.py:13` computes `pg_available` once at conftest import time.
- `tests/conftest.py:14` defines `skip_no_pg = pytest.mark.skipif(...)`.
- `tests/conftest.py:17-18` declares `pg_api` with no explicit `scope=`, so it is function-scoped by default.
- `tests/conftest.py:20-21` raises `pytest.skip` during fixture setup if PostgreSQL is unavailable.
- `tests/conftest.py:23-28` imports `PgApi`, creates an instance, and calls `api.init_db()`.
- `tests/conftest.py:30` yields the API to the test.
- `tests/conftest.py:32-39` is teardown: rollback, delete rows for the unique `domain_id`, commit, and close.

`pyproject.toml:33-34` only sets `testpaths = ["tests"]`; there is no custom fixture/finalization configuration.

Note: the search results claiming zero fixture/yield/skip matches conflict with the actual `tests/conftest.py` content, so treat those search results as unreliable here.

## Pytest finalization rules applied

### 1. Generator fixture lifecycle

For a yield-based fixture, pytest treats:

- code before `yield` as setup,
- the yielded value as the fixture value,
- code after `yield` as teardown/finalization.

Pytest only registers and runs the post-`yield` teardown if the generator actually reached `yield`. If setup raises before `yield`, the code after `yield` does not run.

Applied to `pg_api`:

- If `pytest.skip` at `tests/conftest.py:21` fires, execution never reaches `yield` at line 30. The cleanup at lines 32-39 does not run. That is safe here because no `PgApi` instance was created yet.
- If `api.init_db()` at `tests/conftest.py:28` raises, setup fails before `yield`. The teardown at lines 32-39 does not run, so `api.conn.rollback()`, the deletes, and `api.close()` are skipped. That is a potential resource leak.
- If `yield api` at line 30 is reached, pytest will resume the generator during finalization, even if the test body fails or is skipped later.

### 2. Setup exceptions

If fixture setup raises a normal exception:

- the test is reported as an error, not a failure,
- the test body is not executed,
- fixtures that were already successfully set up are finalized according to their scopes,
- the failing fixture’s own post-`yield` teardown does not run because it never yielded.

If you need cleanup even when setup fails before `yield`, you must use an explicit `try/finally`, `request.addfinalizer(...)` before the failing step, or another cleanup mechanism. The observed `pg_api` fixture does not do that.

Also, the teardown itself is not exception-safe. If `api.conn.rollback()` at `tests/conftest.py:33` or a DELETE at lines 34-38 raises, `api.close()` at line 39 may not execute.

### 3. Skip exceptions

A `pytest.skip(...)` raised during fixture setup is treated as a skip outcome, not a setup failure. Pytest will mark the test as skipped and finalize fixtures that had already completed setup.

Applied to `pg_api`:

- If `DATABASE_URL` is unset, `pytest.skip` at `tests/conftest.py:21` skips the test before `yield`.
- The `pg_api` fixture’s own cleanup at lines 32-39 does not run.
- Any previously set-up fixtures are still finalized.
- If the `skip_no_pg` marker from `tests/conftest.py:14` is applied to a test, pytest can skip the test before setting up `pg_api` at all. The runtime skip at line 21 is the fallback when the fixture is requested without that marker.
- Because `pg_available` is computed once at `tests/conftest.py:13`, changing `DATABASE_URL` after conftest import will not change the skip decision.

### 4. Scoped fixtures

`pg_api` has no `scope=` argument, so it is function-scoped. That means:

- setup runs once per test that requests it,
- each test gets a fresh `PgApi` and a fresh `domain_id` from `uuid.uuid4()` at `tests/conftest.py:26`,
- teardown at lines 32-39 runs after each test, assuming `yield` was reached.

General pytest finalization order is lower-scoped finalizers before higher-scoped finalizers, and within the same scope, reverse setup order. So function-scoped fixtures finalize before class/module/session-scoped fixtures. If a later fixture setup fails after `pg_api` succeeded, `pg_api` still finalizes because its function scope is ending for that test item.

No explicit session/module/class-scoped fixtures were found in the observations. If you later add a session-scoped database fixture, its teardown would run at end of session, not after each test.

## Limitations of the observations

The observations do not show:

- which tests actually use `pg_api`,
- whether tests apply `skip_no_pg`,
- whether any other fixtures exist outside `tests/conftest.py`,
- any `addfinalizer` usage,
- line-numbered source output.

Those details matter if you want to verify finalization behavior across the full suite.

## Topics to Explore

- [file] `tests/conftest.py` — Contains the only observed fixture and demonstrates skip-before-yield and teardown-after-yield behavior.
- [function] `tests/conftest.py:pg_api` — Trace setup, skip, yield, and teardown; note that `init_db()` failure skips cleanup.
- [general] `pytest-fixture-finalization-order` — Learn reverse-order finalization and how scope boundaries affect teardown timing.
- [general] `fixture-setup-exception-cleanup` — Explore `request.addfinalizer` and `try/finally` for cleanup when setup fails before `yield`.
- [general] `skipif-vs-runtime-skip` — Compare marker-based skips with `pytest.skip` inside a fixture setup and their effects on fixture setup/finalization.

## Beliefs

- `pg-api-fixture-is-function-scoped` — `tests/conftest.py:17` uses `@pytest.fixture` without `scope=`, so `pg_api` is function-scoped.
- `skip-before-yield-prevents-pg-api-teardown` — If `DATABASE_URL` is unset, `pytest.skip` at `tests/conftest.py:21` prevents `yield` at line 30, so cleanup at lines 32-39 does not run.
- `init-db-exception-prevents-pg-api-cleanup` — If `api.init_db()` at `tests/conftest.py:28` raises, `api.conn.rollback()` and `api.close()` are not executed because `yield` is never reached.
- `pg-api-teardown-not-exception-safe` — If rollback or DELETE at `tests/conftest.py:33-38` raises, `api.close()` at line 39 is skipped.
- `pg-available-is-cached-at-import` — `pg_available` at `tests/conftest.py:13` is computed once during conftest import, so later `DATABASE_URL` changes do not affect `skip_no_pg` or `pg_api`.
