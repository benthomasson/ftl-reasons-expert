---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Useful for understanding stale-state risks when testing `_db`.

## Explanation

### What `_db` is and how it caches

In `reasons/mcp_server.py`:

- **Line 27** declares a module-level global: `_db: str | None = None`. This is a process-wide cache of the resolved database path.
- **Lines 48–51** form a lazy initializer:
  ```
  global _db
  if _db is None:
      _db = _find_db()
  return _db
  ```
  The critical property is the `if _db is None:` guard — once `_db` is set to any non-`None` value (including a bad path), the getter will **not** re-resolve it. Any subsequent call returns the cached value regardless of what `_find_db()` would now compute.
- **Lines 593–594** are the explicit reset path: `global _db; _db = _find_db()`. This is the only in-module way to force re-computation (likely a `refresh_db()` or similar admin/test hook).

So `_db` is a **write-once-until-reset cache**, and its truth depends entirely on external mutation of the module attribute.

### Why this creates stale-state risk *in tests*

`tests/test_mcp_server.py` mutates the global directly and repeatedly:

- **Line 23**: `mcp_server._db = db_path` (inject a valid path).
- **Line 25**: `mcp_server._db = None` (restore to allow re-lookup).
- **Line 59**: `mcp_server._db = None` (reset at start of another test).
- **Line 65**: `mcp_server._db = None` (reset again).
- **Line 68**: `mcp_server._db = "/some/path.db"` — this looks like the *error path* test: a nonexistent file.
- **Line 70**: `mcp_server._db = None` (reset after).

Risk vectors this exposes:

1. **No autouse fixture / no `monkeypatch.setattr`.** `tests/conftest.py` only defines a `pg_api` fixture (Postgres); there is *nothing* that pins or restores `_db` between tests. The regex search for fixture/monkeypatch patterns around `_db` returned zero matches. Every reset is **inline and manual**, so it's only executed if control flow reaches it.

2. **Failed assertion leaves stale state.** For the ordering at `test_mcp_server.py:68-70`, `_db` is set to `"/some/path.db"` *before* the assertion runs. If that assertion fails (or an exception propagates from the call that consumes `_db`), line 70's reset never executes. The next test inherits a poisoned `_db`, and because of the `if _db is None` guard in `mcp_server.py:49`, the getter will happily return the broken path instead of calling `_find_db()`. Test A's failure silently corrupts Test B.

3. **Test-order dependence.** Because line 23 injects `db_path` and line 25 resets it, running individual tests in isolation vs. as a suite can produce different results. If `pytest -k` runs only the test after line 23 or skips the reset test, the leftover value persists.

4. **Path leakage via `_find_db()`.** If a test resets to `None` but the working directory / env vars have changed (e.g., a prior fixture switched CWD), the *next* call to the lazy getter recomputes `_db` to a different path than the test expects. The reset is only "reset to unknown", not "reset to the value the test had at import time".

5. **Divergent reset mechanisms.** The production code exposes a re-resolve path at `mcp_server.py:593-594`, but the tests bypass it and set `_db` directly. That means the tests test the *attribute*, not the *refresh contract* — so a bug inside `refresh_db()` (line 593-594) would go uncovered.

### Bottom line

`_db` is a process-global cache keyed on "is it `None`?", and `tests/test_mcp_server.py` relies on hand-written reset lines (25, 59, 65, 70) with no fixture-level safety net in `tests/conftest.py`. Any early exit from a test that mutates `_db` (assertion failure, fixture teardown, skipped test, exception) will leak a stale path into the rest of the session, and the `if _db is None` guard in `mcp_server.py:49` will keep serving that stale path instead of recomputing. This is the classic "global state + manual reset + no autouse fixture" pattern that produces test-order-dependent flakes.

## Topics to Explore

- [file] `tests/test_mcp_server.py` — Read lines 15–75 in full to see whether resets are in `setup_method`/`teardown_method` or just inline; the failure-mode analysis hinges on this.
- [function] `reasons/mcp_server.py:_find_db` — Understanding what `_find_db()` reads (env var? CWD? `Path.cwd()`?) determines whether `_db = None` is a safe reset or just a delayed re-resolution of the same stale inputs.
- [function] `reasons/mcp_server.py:refresh_db` — The function at lines 593–594; check its callers and whether any test exercises it, since tests currently bypass it via direct assignment.
- [file] `tests/conftest.py` — Confirmed to lack any `_db` fixture; consider adding an `autouse` fixture with `monkeypatch.setattr(mcp_server, "_db", None)` so parent-test failure cannot leak state.
- [general] `global-cache-test-isolation` — Pattern of searching for other module-level caches (e.g., `_path`, `_root`, `_config`) that may have the same stale-state risk.

## Beliefs

- `mcp-server-_db-is-a-process-global-cache` — `_db` at `reasons/mcp_server.py:27` is a module-level `str | None` that is only recomputed when its value is `None`.
- `mcp-server-getter-short-circuits-on-cached-_db` — The lazy initializer at lines 48–51 returns the cached `_db` without calling `_find_db()` whenever `_db is not None`, so a bad cached path is never corrected by the getter.
- `mcp-server-exposes-refresh-db-path` — Lines 593–594 in `reasons/mcp_server.py` provide an explicit `global _db; _db = _find_db()` reset that tests do not currently use.
- `test-mcp-server-mutates-_db-inline-without-monkeypatch` — Every test touch of `_db` (`tests/test_mcp_server.py:23,25,59,65,68,70`) is a direct attribute assignment; no `monkeypatch.setattr` and no autouse fixture exists.
- `conftest-lacks-db-reset-fixture` — `tests/conftest.py` defines only a `pg_api` fixture (lines 19–40); there is no fixture that resets `mcp_server._db`, so a test that raises after line 23/68 leaves the global poisoned for the remainder of the session.
