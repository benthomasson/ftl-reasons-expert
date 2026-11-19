---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_pg_dispatch.py

# `tests/test_pg_dispatch.py`

## Purpose

This file is a **unit test suite for the PostgreSQL dispatch layer** — the plumbing that lets `reasons.api` / `reasons.cli` route operations to PostgreSQL (`reasons.pg.PgApi`) instead of the default SQLite store whenever a `pg_conninfo` + `domain_id` pair is supplied.

It owns three responsibilities:

1. **Verify the dispatch helper** `_pg_dispatch` correctly instantiates `PgApi`, forwards kwargs, and returns its result.
2. **Verify the CLI backend-selection helpers** `_backend_kwargs` and `_require_sqlite` from `reasons.cli` (precedence rules, validation, gating).
3. **Regression-test that many previously SQLite-only commands now accept PostgreSQL** — the `*NoLongerBlocked` classes are explicit signposts that `import_json`, `import_beliefs`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, and `list_negative` were once guarded by `_require_sqlite` and now dispatch to `PgApi`.

It is **not** an integration test: every `PgApi` call is mocked, so the suite exercises the *plumbing contract* (which method is called, with which kwargs, and what result flows back), not database behavior.

---

## Key Components

### Helper under test: `_pg_dispatch(conninfo, domain_id, method, **kwargs)` (from `reasons.api`)
- Opens `PgApi` **as a context manager** (`with PgApi(...) as pg:`).
- Calls `getattr(pg, method)(**kwargs)` and returns the value.
- Contract enforced by tests:
  - Return value is passed through unchanged (`test_dispatch_calls_pgapi_method`).
  - All kwargs are forwarded verbatim (`test_dispatch_passes_kwargs` asserts `show_node.assert_called_once_with(node_id="a")`).
  - Exceptions from the `with` body are **not** swallowed (`__exit__` is mocked to return `False`).

### CLI helpers under test (from `reasons.cli`)

**`_backend_kwargs(args)`**
- Returns the dict consumed downstream by storage/API factories.
- Outcomes:
  - No PG anywhere → `{"db_path": "reasons.db"}` (`args.db`).
  - PG selected → `{"pg_conninfo": <conninfo>, "domain_id": <project id>}`.
- **Precedence: CLI args > env vars** (`REASONS_PG_CONNINFO`, `REASONS_PROJECT_ID`).
- If PG is selected but no `domain_id` can be resolved → raises `SystemExit`.

**`_require_sqlite(args, operation_name)`**
- A **gate** for commands that must run against SQLite.
- Raises `SystemExit` if `args.pg` is set **or** `REASONS_PG_CONNINFO` is set in the environment; otherwise returns `None`.
- Second argument (e.g. `"hash-sources"`) is a label used in the error message.

### `EXPORT_DATA` (constant inside `TestExportMarkdownPgPath`)
A representative `export_network` payload with four nodes (`premise-a`, `derived-b`, `gated-c`, `blocker`), one nogood, and metadata. It exercises:
- a source-bearing premise (`premise-a`),
- a derived node (`derived-b`) whose justification points at `premise-a`,
- an OUT/gated node (`gated-c`) with an `outlist`,
- a nogood referencing two nodes.

The markdown tests patch `reasons.api.export_network` to return this dict, so `export_markdown(pg_conninfo=..., domain_id=...)` is being tested purely as a **renderer** over data fetched elsewhere.

### Per-command dispatch test classes
Each class follows the same shape — one test for the default kwargs, one for a non-default kwarg path, plus result-shape assertions:

| Class | Method asserted on `PgApi` | Default kwargs verified |
|---|---|---|
| `TestImportJsonDispatch` | `import_json(data=<parsed>)` | parses file → `data=` |
| `TestImportBeliefsDispatch` | `import_beliefs(beliefs_text=<text>)` | passes raw text |
| `TestImportAgentDispatch` | `import_agent(agent_name=..., claims=[...])` | markdown and JSON inputs |
| `TestSyncAgentDispatch` | `sync_agent(...)` | called once |
| `TestHashSourcesDispatch` | `hash_sources(force=False, repos=None)` | `force` / `repos` forwarded |
| `TestCheckStaleDispatch` | `check_stale(repos=None, upgrade_hashes=False, git_aware=False)` | `upgrade_hashes=True` forwarded |
| `TestLookupDispatch` | `lookup(query=..., visible_to=None, include_out=False)` | `visible_to` forwarded |
| `TestAddRepoDispatch` | `add_repo(name=..., path=...)` | kwargs forwarded |
| `TestListReposDispatch` | `list_repos()` — **no kwargs** | empty result OK |
| `TestListNegativeDispatch` | `list_negative(visible_to=None, model="claude", skip_llm=False)` | `visible_to` + `model` forwarded |

---

## Patterns

### 1. Context-manager mocking of `PgApi`
Every test repeats this idiom:

```python
with patch("reasons.pg.PgApi") as MockPgApi:
    MockPgApi.return_value.__enter__ = MagicMock(return_value=mock_pg)
    MockPgApi.return_value.__exit__  = MagicMock(return_value=False)
    result = <api fn>(pg_conninfo="postgresql://...", domain_id="test")
```

- `__enter__` returns the fake `PgApi` instance so `with PgApi(...) as pg:` binds `pg` to `mock_pg`.
- `__exit__` returns **`False`** — the standard “do not suppress exceptions” sentinel. This matters: if `_pg_dispatch` raises inside the `with`, the test would still see it.

### 2. Patching at the definition site, not the import site
The patch target is `reasons.pg.PgApi`, **not** `reasons.api.PgApi`. This is a strong hint about implementation: `reasons.api` must resolve `PgApi` through the `reasons.pg` module at call time (e.g. `import reasons.pg as pg` then `pg.PgApi(...)`, or a deferred import inside `_pg_dispatch`). Otherwise the patch would have no effect.

### 3. Environment isolation with `patch.dict(..., clear=True)`
Env-sensitive tests strip `REASONS_PG_CONNINFO` / `REASONS_PROJECT_ID` from `os.environ` and re-inject them with `clear=True`, so results don't depend on the developer's shell.

### 4. “`*NoLongerBlocked`” regression classes
Named after the behavior change (`_require_sqlite` gating removed for that command). They duplicate a dispatch assertion from the primary class but frame it as “this used to raise `SystemExit`, now it returns a result.” Useful anchors for future refactors.

### 5. Arrange → call → assert-call → assert-result
Most tests assert **both** forwarding (`mock_pg.X.assert_called_once_with(...)`) and passthrough (`assert result[...] == ...`), catching regressions in either direction.

---

## Dependencies

**Imports (test → code):**
- `reasons.api`: `_pg_dispatch`, `export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`.
- `reasons.cli`: `_backend_kwargs`, `_require_sqlite`.
- `reasons.pg.PgApi` (indirect — via `patch`).

**Under test (implicit contracts):**
- `reasons/pg.py` — the `PgApi` class must be a context manager exposing all the methods listed above.
- `reasons/api.py` — functions must accept `pg_conninfo=` + `domain_id=` kwargs and prefer them over `db_path`.
- `reasons/cli.py` — argument names `args.pg`, `args.domain_id`, `args.db`, and env `REASONS_PG_CONNINFO`, `REASONS_PROJECT_ID`.

**Sibling test:** `tests/test_pg.py` (likely tests `PgApi` internals — this file only tests the *dispatch to* `PgApi`).

Note the repository also contains a parallel `reasons_lib/` package; these tests import from `reasons.*`, so they exercise that package tree.

---

## Flow

### Dispatch flow (e.g. `import_json`)
1. Test writes a JSON file in `tmp_path`.
2. Calls `import_json(path, pg_conninfo=..., domain_id="test")`.
3. The function detects PG mode, reads the file, `json.loads` it.
4. Calls `_pg_dispatch(conninfo, domain_id, "import_json", data=parsed)`.
5. `_pg_dispatch` enters `PgApi(...)`, calls `pg.import_json(data=parsed)`, returns the dict.
6. Test asserts forwarded kwargs and `result["nodes_imported"]`.

### `export_markdown` flow
1. Test patches `reasons.api.export_network` to return `EXPORT_DATA`.
2. Calls `export_markdown(pg_conninfo=..., domain_id=...)`.
3. `export_markdown` fetches the network dict (mocked), renders dependents from justifications (e.g. `premise-a` is an antecedent of `derived-b` / `gated-c`), and emits a markdown string.
4. Tests scan the string for node IDs, text, source paths, and nogood references — i.e. they check **rendering completeness**, not rendering exactness.

### `_backend_kwargs` flow
1. Resolve conninfo: `args.pg` → else `REASONS_PG_CONNINFO`.
2. Resolve domain id: `args.domain_id` → else `REASONS_PROJECT_ID`.
3. If conninfo is set but domain id is missing → `SystemExit`.
4. Return either `{"db_path": args.db}` or `{"pg_conninfo": ..., "domain_id": ...}`.

### `_require_sqlite` flow
1. If `args.pg` **or** `REASONS_PG_CONNINFO` → `SystemExit`.
2. Otherwise return (implicit `None`).

---

## Invariants

- **`_pg_dispatch` is context-managed.** Tests set up `__enter__`/`__exit__`; removing the `with` from the implementation would break every dispatch test.
- **CLI args beat env vars** in `_backend_kwargs` (`test_cli_overrides_env`).
- **Env vars beat the SQLite default** (`test_env_var_fallback`); but env vars must be *cleared* to fall through to `{"db_path": ...}`.
- **PG requires a domain id.** Missing on both CLI and env → `SystemExit` (`test_pg_missing_domain_id_exits`).
- **`_require_sqlite` is a hard gate**, not advisory — both `args.pg` and `REASONS_PG_CONNINFO` trigger `SystemExit`.
- **Filenames are validated before dispatch.** `import_json("/nonexistent.json", pg_conninfo=...)` raises `FileNotFoundError` without ever touching `PgApi`.
- **Default kwargs are passed explicitly, never omitted.** The assertions pin `hash_sources(force=False, repos=None)`, `check_stale(repos=None, upgrade_hashes=False, git_aware=False)`, `lookup(query=..., visible_to=None, include_out=False)`, and `list_negative(visible_to=None, model="claude", skip_llm=False)`. Changing a default signature will fail these tests.
- **`list_repos` takes no kwargs** — the assertion is `mock_pg.list_repos.assert_called_once_with()`.
- **`import_json` and `import_beliefs` forward parsed/raw content via `data=` and `beliefs_text=`** respectively — not via positional args.

---

## Error Handling

- **`SystemExit`** is the expected failure mode for CLI-level misconfiguration:
  - `_backend_kwargs` on PG-without-domain (`test_pg_missing_domain_id_exits`).
  - `_require_sqlite` on PG-configured environment (`test_pg_set_exits`, `test_env_pg_exits`).
- **`FileNotFoundError`** propagates from `import_json` before dispatch (`test_file_not_found`), i.e. file I/O errors are *not* swallowed by the PG branch.
- **Exceptions from inside `PgApi`** propagate: `__exit__` mock returns `False`, so any real error raised by a `PgApi` method surfaces to the caller rather than being masked.
- No other error branches are exercised; the tests do **not** verify what happens when `PgApi` raises (e.g. `psycopg.OperationalError`) — that's presumably covered in `test_pg.py` or an integration suite.

---

## Topics to Explore

- [file] `reasons/pg.py` — The `PgApi` implementation whose methods this entire suite mocks; confirms the context-manager protocol and method signatures the tests rely on.
- [function] `reasons/api.py:_pg_dispatch` — The actual dispatcher; confirms the deferred `PgApi` lookup that makes patching `reasons.pg.PgApi` work.
- [function] `reasons/cli.py:_backend_kwargs` — The precedence/validation logic verified by `TestBackendKwargs`; check the exact `SystemExit` message and env-var names.
- [file] `tests/test_pg.py` — Sibling unit tests for `PgApi` itself (SQL/schema assertions) that this dispatch suite deliberately avoids.
- [general] `pg-vs-sqlite-backend-parity` — The broader migration so many operations could drop `_require_sqlite`; explains the `*NoLongerBlocked` test classes and which commands remain SQLite-only.

## Beliefs

- `pg-dispatch-is-context-managed` — `_pg_dispatch` instantiates `PgApi` as a context manager and delegates the named method with forwarded kwargs, returning its result unchanged.
- `backend-kwargs-cli-overrides-env` — `_backend_kwargs` gives CLI args (`args.pg`, `args.domain_id`) precedence over `REASONS_PG_CONNINFO` / `REASONS_PROJECT_ID`.
- `pg-requires-domain-id` — `_backend_kwargs` raises `SystemExit` when PostgreSQL is selected without a resolvable domain/project id, whereas `_require_sqlite` raises `SystemExit` whenever PostgreSQL is configured at all.
- `pgapi-patched-via-module-namespace` — Every test patches `reasons.pg.PgApi` (not `reasons.api.PgApi`), implying `reasons.api` resolves `PgApi` through the `reasons.pg` module at call time rather than as a module-level import binding.
- `import-json-parses-before-dispatch` — `import_json` raises `FileNotFoundError` and parses JSON into `data=` before ever invoking `PgApi.import_json`, so no DB session is opened on a missing/invalid file path.
