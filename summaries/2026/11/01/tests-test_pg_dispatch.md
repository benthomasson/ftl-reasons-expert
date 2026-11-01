---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_pg_dispatch.py

# `tests/test_pg_dispatch.py` — PostgreSQL Dispatch Layer Tests

## Purpose

This file verifies the **backend routing logic** that sits between the public API functions (`reasons.api`) and the two storage backends (SQLite via `reasons.storage` vs. PostgreSQL via `reasons.pg.PgApi`). Its responsibility is to ensure that:

1. When `pg_conninfo` + `domain_id` are supplied, every supported API function delegates to the correct `PgApi` method with the correct arguments.
2. The CLI argument-parsing helper `_backend_kwargs` correctly resolves which backend to use (CLI flags override env vars; SQLite is the default).
3. Commands that are **SQLite-only** (e.g., `hash-sources` in older versions) still guard against PG mode via `_require_sqlite`.
4. A set of "NoLongerBlocked" regression tests confirm that import/maintenance/repo commands, which were previously hard-blocked from PG, now correctly dispatch to the PG path.

In short, it is the contract test for the **dual-backend dispatch seam**.

## Key Components

### `TestPgDispatch`
Tests the core `_pg_dispatch(conninfo, domain_id, method_name, **kwargs)` helper from `reasons.api`.

| Test | Contract verified |
|---|---|
| `test_dispatch_calls_pg_method` | `_pg_dispatch` instantiates `PgApi(conninfo, domain_id)` as a **context manager**, calls `get_status()`, and returns its result. |
| `test_dispatch_passes_kwargs` | Arbitrary keyword args (e.g., `node_id="a"`) are forwarded verbatim to the target method (`show_node`). |

### `TestBackendKwargs`
Tests `_backend_kwargs(args: argparse.Namespace) -> dict` from `reasons.cli`.

| Test | Invariant |
|---|---|
| `test_sqlite_default` | No `--pg` flag and no `REASONS_PG_CONNINFO` env → returns `{"db_path": ...}` (SQLite). |
| `test_pg_with_domain_id` | Both `--pg` and `--domain-id` present → returns `{"pg_conninfo": ..., "domain_id": ...}`. |
| `test_pg_missing_domain_id_exits` | `--pg` set but `--domain-id` is `None` and no env fallback → raises `SystemExit`. |
| `test_env_var_fallback` | `REASONS_PG_CONNINFO` / `REASONS_PROJECT_ID` env vars are used when CLI flags are absent. |
| `test_cli_overrides_env` | CLI flags take precedence over env vars. |

### `TestRequireSqlite`
Tests `_require_sqlite(args, command_name)`:

- `test_no_pg_passes` — no PG indicator → function returns silently.
- `test_pg_set_exits` — `args.pg` is set → raises `SystemExit` (the command is SQLite-only).
- `test_env_pg_exits` — `REASONS_PG_CONNINFO` env var is set → also raises `SystemExit`.

### `TestExportMarkdownPgPath`
A rich integration-style test (still unit-level via mocks). It provides a realistic `EXPORT_DATA` dict (4 nodes: premise, derived, gated, blocker; one nogood) and verifies that `export_markdown(pg_conninfo=..., domain_id=...)` produces a non-empty markdown string containing node IDs, node text, source info, nogood references, and reconstructed dependents.

### Dispatch Test Classes (one per API function)
Each follows the same template:

```
TestImportJsonDispatch
TestImportBeliefsDispatch
TestImportAgentDispatch
TestSyncAgentDispatch
TestHashSourcesDispatch
TestCheckStaleDispatch
TestLookupDispatch
TestAddRepoDispatch
TestListReposDispatch
TestListNegativeDispatch
```

Each class asserts two things:
1. **The correct `PgApi` method is called** with the expected keyword arguments (e.g., `mock_pg.import_json.assert_called_once_with(data=data)`).
2. **The return value is passed through** to the caller unchanged.

### Regression / "NoLongerBlocked" Classes
`TestImportCliNoLongerBlocked`, `TestMaintenanceCliNoLongerBlocked`, `TestRepoAndNegativeCliNoLongerBlocked` — these are **regression guards**. They exist because a previous version of the code had an explicit check that raised `SystemExit` if these commands were invoked with PG credentials. These tests confirm that guard has been removed and the PG path is now functional.

## Patterns

- **Context-manager mocking idiom.** Every PG dispatch test repeats:
  ```python
  MockPgApi.return_value.__enter__ = MagicMock(return_value=mock_pg)
  MockPgApi.return_value.__exit__ = MagicMock(return_value=False)
  ```
  This tells us `PgApi` is used as a context manager (`with PgApi(...) as pg:`) inside `_pg_dispatch`. The `__exit__` returning `False` means exceptions are *not* suppressed.

- **`patch.dict("os.environ", ..., clear=True)`** — the `clear=True` flag is critical. It wipes *all* environment variables before applying the test-specific ones, preventing CI environment noise (e.g., a real `REASONS_PG_CONNINFO` set in CI) from leaking in.

- **`argparse.Namespace` as a stand-in for parsed CLI args.** The tests never invoke the real `argparse` parser; they construct the namespace directly. This decouples the tests from the CLI schema.

- **Table-driven regression tests.** The "NoLongerBlocked" classes are near-identical to the primary dispatch tests but were added after a behavior change. They exist purely to prevent the old guard from reappearing.

- **Shared fixture via class attribute.** `TestExportMarkdownPgPath.EXPORT_DATA` is a class-level dict used across multiple test methods, avoiding repetition.

## Dependencies

### Imports (this file → project)
| Import | Used for |
|---|---|
| `reasons.api._pg_dispatch` | Core dispatch helper under test |
| `reasons.api.export_markdown, import_json, import_beliefs, import_agent, sync_agent, hash_sources, check_stale, lookup, add_repo, list_repos, list_negative` | Public API functions whose PG paths are exercised |
| `reasons.cli._backend_kwargs` | CLI arg → backend config resolution |
| `reasons.cli._require_sqlite` | SQLite-only guard |
| `reasons.pg.PgApi` (via `patch`) | The mock target; represents the PG backend |

### Imported by (project → this file)
None directly. This is a leaf test file. It is discovered and run by `pytest` as part of the `tests/` suite. It is implicitly depended upon by CI (`.github/workflows/`) as a gate for any change touching the dispatch seam.

## Flow

Every test in this file follows one of two execution shapes:

### Shape A — Direct API call with PG kwargs
```
Test method
  → constructs test inputs (file path, string, dict, etc.)
  → patches reasons.pg.PgApi with a MagicMock context manager
  → calls the public API function with pg_conninfo + domain_id
  → inside the function, _pg_dispatch:
       1. instantiates PgApi(conninfo, domain_id)
       2. enters the context manager (yields mock_pg)
       3. calls getattr(mock_pg, method_name)(**kwargs)
       4. exits the context manager
  → test asserts mock was called with expected args
  → test asserts return value matches mock's return_value
```

### Shape B — CLI config resolution
```
Test method
  → builds argparse.Namespace
  → patches os.environ (clearing first)
  → calls _backend_kwargs(args)
  → asserts the returned dict selects the correct backend
  → (for _require_sqlite) asserts SystemExit or silent pass
```

There is no async, no network I/O, and no real database. All PG interaction is mocked at the `PgApi` class boundary.

## Invariants

1. **CLI overrides env.** `_backend_kwargs` must prefer `args.pg` / `args.domain_id` over `REASONS_PG_CONNINFO` / `REASONS_PROJECT_ID`. (`test_cli_overrides_env`)
2. **PG requires `domain_id`.** If `pg_conninfo` is present but `domain_id` is not resolvable (neither from CLI nor env), the process must exit. (`test_pg_missing_domain_id_exits`)
3. **SQLite is the default.** Absence of any PG indicator yields `{"db_path": ...}`. (`test_sqlite_default`)
4. **`_require_sqlite` guards SQLite-only commands.** If PG is configured (via CLI *or* env), the command must exit. (`test_pg_set_exits`, `test_env_pg_exits`)
5. **Dispatch is a transparent pass-through.** The return value of the `PgApi` method is returned unchanged to the caller. (`test_dispatch_calls_pg_method` and all other dispatch tests)
6. **`PgApi` is a context manager.** The dispatch code uses `with PgApi(...) as pg:`, so `__enter__`/`__exit__` must be present. (Every mock setup includes both.)

## Error Handling

| Scenario | Behavior | Where tested |
|---|---|---|
| `domain_id` missing with PG configured | `SystemExit` (raised by `_backend_kwargs`) | `test_pg_missing_domain_id_exits` |
| PG configured but command is SQLite-only | `SystemExit` (raised by `_require_sqlite`) | `test_pg_set_exits`, `test_env_pg_exits` |
| Input file does not exist | `FileNotFoundError` (raised by `import_json` before dispatch) | `test_file_not_found` |
| PG call raises inside context manager | `__exit__` returns `False`, so the exception propagates. Not explicitly tested here, but implied by the mock setup. | — |
| Empty / edge-case data (empty network, empty repos) | Functions return valid structures; no crash | `test_empty_network`, `test_empty_repos` |

Notably, there is **no test for PG connection failure** (e.g., `PgApi.__enter__` raising `OperationalError`). That failure mode lives in `reasons/pg.py` and would be covered by `tests/test_pg.py` if at all.

## Topics to Explore

- [file] `reasons/api.py` — The dispatch seam itself: `_pg_dispatch`, and each public function's `if pg_conninfo: ... else: ...` branching. Understanding the production code is prerequisite to interpreting these tests.
- [file] `reasons/pg.py` — The `PgApi` class and its context-manager protocol. The tests mock this boundary; reading the real implementation reveals what SQL queries and transaction semantics sit behind each method.
- [file] `reasons/cli.py` — The `argparse` setup, `_backend_kwargs`, and `_require_sqlite`. Shows how CLI flags and env vars are wired together before the API layer is invoked.
- [file] `tests/test_pg.py` — Tests for the `PgApi` class itself (connection pooling, SQL correctness, transaction handling). Complements this file, which only verifies the *routing* to `PgApi`.
- [general] `sqlite-vs-pg-parity` — Which API functions support both backends vs. which are still SQLite-only. The "NoLongerBlocked" classes hint at a gradual migration; checking the current CLI help or CHANGELOG reveals what's been unblocked.

## Beliefs

- `pg-dispatch-requires-domain-id` — `_backend_kwargs` raises `SystemExit` when `pg_conninfo` is resolvable but `domain_id` is `None` from both CLI and environment.
- `cli-flags-override-env-vars` — In `_backend_kwargs`, `args.pg` and `args.domain_id` take precedence over `REASONS_PG_CONNINFO` and `REASONS_PROJECT_ID`; env vars are only consulted when the corresponding CLI flag is absent.
- `pgapi-is-a-context-manager` — `_pg_dispatch` uses `with PgApi(conninfo, domain_id) as pg:`, meaning `PgApi` must implement `__enter__`/`__exit__`; all tests in this file mock both methods explicitly.
- `require-sqlite-guards-against-pg-env` — `_require_sqlite` checks *both* `args.pg` and the `REASONS_PG_CONNINFO` environment variable, so a command can be blocked from PG mode via either channel.
- `export-markdown-pg-path-produces-identical-output` — `export_markdown` called with `pg_conninfo` delegates to `export_network` (mocked here) and then renders the same markdown structure as the SQLite path; the tests verify node IDs, text, justifications, sources, nogoods, and dependents are all present in the output.
