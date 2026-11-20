---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Review pytest's generator-fixture semantics (setup/yield/teardown, function vs module scope, interaction with `pytest.skip` in setup) to fully understand the failure modes described above.

## Pytest generator-fixture semantics

A `@pytest.fixture` function that contains `yield` is a generator fixture:

```python
@pytest.fixture
def fx():
    # setup
    resource = make_resource()
    yield resource
    # teardown
    resource.close()
```

Semantics:

- Code **before** `yield` is setup.
- The value passed to `yield` is the fixture value injected into tests.
- Code **after** `yield` is teardown/finalization.
- Teardown runs if setup completed and `yield` was reached, even if the test itself fails or errors.
- If setup raises before `yield`, the post-`yield` teardown does **not** run. Any partial resources created before the exception must be cleaned up explicitly with `try/finally`, `request.addfinalizer`, or equivalent.
- `pytest.skip()` in setup raises a skip exception. If this happens before `yield`, the requesting test is skipped and the generator fixture’s post-`yield` teardown is not executed.

Scope matters:

- Default fixture scope is `function`, so setup/teardown happen once per test.
- A `module`-scoped generator fixture is set up once per module and torn down after the last test in that module that uses it.
- If a module-scoped fixture calls `pytest.skip()` during setup, all tests in that module that depend on that fixture are skipped. If the fixture is autouse and module-scoped, that effectively skips the module’s tests that rely on it.
- Higher-scoped fixtures cannot depend on lower-scoped fixtures; e.g. a module-scoped fixture cannot depend on a function-scoped fixture.

## Applied to `tests/conftest.py`

The observed fixture is `tests/conftest.py:17-39`:

- `tests/conftest.py:13` computes `pg_available = bool(os.environ.get("DATABASE_URL"))` at import time.
- `tests/conftest.py:14` defines `skip_no_pg = pytest.mark.skipif(...)`, but this mark is not applied inside `conftest.py`.
- `tests/conftest.py:17-18` defines `pg_api` with no `scope=` argument, so it is **function-scoped** by default.
- `tests/conftest.py:20-21` calls `pytest.skip("DATABASE_URL not set")` during fixture setup if `pg_available` is false.
- `tests/conftest.py:23-28` imports `PgApi`, creates a unique `domain_id`, constructs `PgApi`, and calls `api.init_db()`.
- `tests/conftest.py:30` yields the `api` object to the test.
- `tests/conftest.py:32-39` runs teardown: rollback, delete rows for `domain_id`, commit, and `api.close()`.

So the normal path is:

1. Test requests `pg_api`.
2. Pytest checks `pg_available`.
3. If false, `pytest.skip` at line 21 raises during setup; test is skipped; no teardown after line 30 runs.
4. If true, `PgApi` is created and initialized.
5. Test runs with `api`.
6. Teardown lines 32-39 run after the test, including when the test fails.

The `skip_no_pg` mark at line 14 is a separate mechanism. If tests use `@skip_no_pg`, pytest skips at selection/mark time and may not even set up `pg_api`. The fixture’s runtime `pytest.skip` is a second safety net.

## Likely failure modes

- **Skip-before-yield does not run teardown.**  
  This is intentional for the no-`DATABASE_URL` case because `PgApi` is not created before the skip. But it means any future allocation placed before line 21 would leak unless guarded.

- **`api.init_db()` failure can leak resources.**  
  `PgApi(...)` is created at `tests/conftest.py:27`, and `api.init_db()` is called at line 28 before `yield`. If `init_db()` raises, `yield` is not reached and the teardown at lines 32-39 does not run. If `PgApi` opens a connection in its constructor, that connection may not be closed.

- **Teardown is not exception-safe.**  
  Lines 32-39 run sequentially without `try/finally`. If `rollback()` fails, or a `DELETE` fails, or `commit()` fails, later cleanup such as `api.close()` may not run. A teardown failure can also mask the original test failure.

- **`pg_available` is captured at import time.**  
  Because `pg_available` is computed at `tests/conftest.py:13`, later environment changes do not update it. If `DATABASE_URL` is removed after import, `pg_available` may still be true and line 25 can raise `KeyError`.

- **Function scope gives per-test isolation.**  
  Each test gets a fresh `PgApi` and unique `domain_id`, and cleanup is scoped to that `domain_id`. If `pg_api` were changed to module scope, tests would share one `api` and one `domain_id`; cleanup would happen only after the module, and test order/interference would become significant.

## Reliability of the observations

The search observations are internally inconsistent:

- `find_pytest_skip_usage` reports zero matches, but `tests/conftest.py:21` contains `pytest.skip`.
- `find_generator_fixtures_yield` reports zero matches, but `tests/conftest.py:30` contains `yield`.
- `find_all_fixture_decorators` reports zero matches, but `tests/conftest.py:17` contains `@pytest.fixture`.

The `find_module_scoped_fixtures` zero result is consistent with the file, but the other zero results mean the search results should not be trusted without re-running them over `tests/conftest.py`.

`pyproject.toml:33-34` only sets `testpaths = ["tests"]`. There is no observed pytest configuration enforcing a fixture scope or adding `addopts`.

## What’s missing

The observations do not include the actual failing test output, which tests request `pg_api`, whether `skip_no_pg` is used elsewhere, or the implementation of `reasons.pg.PgApi`. Those details are needed to map these semantics to the exact failure modes “described above.”

## Topics to Explore

- [file] `tests/conftest.py` — Confirm which tests request `pg_api` and whether `skip_no_pg` is actually used outside this file.
- [function] `tests/conftest.py:pg_api` — The central generator fixture whose skip, yield, and teardown behavior determines the test failure modes.
- [general] `pytest-generator-fixture-semantics` — Explore pytest’s finalization rules for setup exceptions, skip exceptions, and scoped fixtures.
- [general] `pg-api-lifecycle` — Inspect `reasons.pg.PgApi` to see whether constructor or `init_db()` failures leave open connections/resources.
- [file] `pyproject.toml` — Verify there are no hidden pytest `addopts`, scope settings, or plugin configurations beyond `testpaths`.

## Beliefs

- `pg-api-is-function-scoped` — `pg_api` has no `scope=` argument, so pytest creates and tears it down once per requesting test.
- `pg-api-skips-before-yield-when-database-url-missing` — If `DATABASE_URL` is absent at conftest import, `pytest.skip` at `tests/conftest.py:21` raises before `yield`, so post-yield cleanup does not run.
- `pg-api-teardown-is-not-exception-safe` — Cleanup at `tests/conftest.py:32-39` runs without `try/finally`, so a failure in rollback/delete/commit can prevent `api.close()`.
- `pg-availability-is-captured-at-import-time` — `pg_available` at `tests/conftest.py:13` and `skip_no_pg` at line 14 are evaluated once when `conftest.py` is imported.
- `search-results-contradict-conftest-content` — Reported zero matches for `@pytest.fixture`, `yield`, and `pytest.skip` conflict with `tests/conftest.py:17`, `:21`, and `:30`.
