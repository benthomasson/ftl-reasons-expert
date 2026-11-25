---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Compare `PgApi` methods against the 19 still-gated commands to identify what porting work remains.

# Comparing `PgApi` against the still-gated commands

## What we can see, and what "gated" means here

There are two different uses of "gated" in this codebase, and they must not be confused:

1. **Domain "gating"** — the TMS concept of a belief whose `SL` justification carries an `outlist` blocker. This is what `api.list_gated` (`reasons/api.py:3628`) and `api.report_gated` (`reasons/api.py:3671`) are about, plus the whole `reasons/report_gated.py` module. It has nothing to do with porting.
2. **Backend "gating"** — a CLI command that still refuses to run against Postgres because no `PgApi` method exists for it. This is enforced by `_require_sqlite(args, command_name)` in `reasons/cli.py`, exercised in `tests/test_pg_dispatch.py::TestRequireSqlite`. `hash-sources` is the one concrete example shown (`test_pg_set_exits`).

The task ("19 still-gated commands") is about the second meaning. **The observations do not contain the list of 19**, so I can only describe the method for the comparison and the one or two data points we can actually verify.

## The dispatch contract

`reasons/api.py` exposes a `_pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs)` helper. We see:

- One call site: `api.py:3639` — `return _pg_dispatch(pg_conninfo, domain_id, "list_gated", ...)`.
- Its behaviour from `tests/test_pg_dispatch.py::TestPgDispatch`:
  - patches `reasons.pg.PgApi`, enters it as a context manager,
  - calls `getattr(pg, method_name)(**kwargs)` and returns the result.

So the *implicit contract* is:

> For every API function that accepts `pg_conninfo`/`domain_id`, there must be a matching `PgApi.<method_name>` on the `PgApi` class in `reasons/pg.py`, accepting the same keyword arguments, and returning the same shape as the SQLite path.

Any API function whose `pg_conninfo` path calls `_require_sqlite` (or does not dispatch at all) is "still gated."

## What the code currently tells us about porting status

| Command | Evidence | Status |
|---|---|---|
| `list_gated` | `api.py:3639` dispatches to `"list_gated"` | Ported (method exists somewhere in the unseen ~3.5k lines of `pg.py`) |
| `report_gated` | `api.py:3671` calls `_report_gated(...)` but at `3708` invokes `list_gated(**vis, **backend)` and re-uses `gated_data` | Likely **composed** — may not need its own `PgApi` method if `list_gated` + `show`/count primitives are ported. Worth confirming whether `report_gated` itself calls `_pg_dispatch`. |
| `export_markdown` | `tests/test_pg_dispatch.py::TestExportMarkdownPgPath` calls `export_markdown(pg_conninfo=..., domain_id=...)` and patches `export_network` — so `export_markdown` is a wrapper over `export_network` and is PG-routed | Ported at the API layer |
| `hash-sources` | `TestRequireSqlite::test_pg_set_exits` asserts a `SystemExit` when `--pg` is set; `test_env_pg_exits` for the env var | **Still gated** |
| `get_status`, `show_node` | `TestPgDispatch` uses them as the canonical dispatch examples | Ported |
| `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative` | Imported by `tests/test_pg_dispatch.py` | Mixed — the visible 200 lines only exercise dispatch and `export_markdown`; the remaining 373 lines of the test file would tell us which are gated. |

This gives us maybe 3 of the 19 gated commands identified by name. The rest are not in the observations.

## What the actual porting work looks like, per command

From the schema and patterns in `reasons/pg.py` (`SCHEMA` at lines ~20–68, `INDEXES` at ~71–85, `_add_node_raw` at ~207), the port of a gated command requires all of:

1. **Add `PgApi.<name>`** in `reasons/pg.py`. Follow the two-tier pattern:
   - `_raw` helpers (like `_add_node_raw`) take a cursor, do **not** commit, and let the caller own the transaction.
   - Public methods (like `init_db`, lines ~152–205) open `self.conn.cursor()`, run the SQL, then `self.conn.commit()`.
2. **Respect `self.domain_id`** — every row in `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, and `rms_network_meta` is keyed by `domain_id` (see PKs in `SCHEMA`). Every query must be scoped, otherwise multi-tenant isolation breaks.
3. **Match the SQLite result shape** (e.g., `list_gated` must return `{"blockers": {...}, "gated_count": int, "blocker_count": int}` per `api.py:3635–3636`, or `report_gated` at `api.py:3726` will break).
4. **Add a `_pg_dispatch` branch** in the corresponding `reasons/api.py` function, matching the `list_gated` example at `api.py:3639`.
5. **Remove the `_require_sqlite(args, "<name>")` guard** in `reasons/cli.py` (the pattern tested by `TestRequireSqlite`).
6. **Confirm `_backend_kwargs`** (tested in `TestBackendKwargs`) now emits `{"pg_conninfo": ..., "domain_id": ...}` — it already does for `list_gated`-style commands.

### Subtlety: composed commands and transaction boundaries

`_pg_dispatch` uses `with PgApi(...)` (per `TestPgDispatch`), and `PgApi.__exit__` rolls back only on exception; `close()` closes the connection. That means **each dispatch call is its own transaction**. For a composed command like `report_gated`, which internally calls `list_gated` plus several `show`/count lookups (`api.py:3708–3721`), each backend call gets a *separate* connection/transaction. Porting composed commands therefore isn't just "add a method" — you have to decide whether to accept non-snapshot consistency or introduce a batched `PgApi` method that returns everything in one transaction.

## What is missing to complete the comparison

The task as stated cannot be finished from the observations:

- **`reasons/pg.py` is 3,736 lines** but only the first 200 are shown (through `_add_node_raw`). We see `init_db` and one raw helper; the other ~3.5k lines — where every ported method lives — are invisible. We cannot enumerate `PgApi`'s actual method set.
- **No list of the 19 gated commands.** The search `still.?gated|gated command|gated_commands|GATED|gated` returned 235 hits, but all sampled hits are about TMS blocking (`list_gated`, `report_gated`, `blockers`, `gated_count`). The mechanism that marks a *CLI* command as gated (`_require_sqlite`) never appeared in the search results — it needs `grep -n "_require_sqlite" reasons/cli.py`.
- **`_pg_dispatch` implementation** in `api.py` is not shown, only callers and tests.
- **The full `tests/test_pg_dispatch.py`** (only 200/573 lines shown) likely contains the de-facto inventory of which commands are PG-routed vs. gated.

A reliable script for the comparison would be:

```
for cmd in $(grep -oP '_require_sqlite\([^,]+,\s*"\K[^"]+' reasons/cli.py); do
    echo "GATED: $cmd"
done
# then for each, check:
grep -n "_pg_dispatch(.*\"$cmd\"" reasons/api.py
grep -n "def ${cmd//-/_}\b" reasons/pg.py
```

Anything in the first list with no match in either of the other two is the remaining porting workload.

## Topics to Explore

- [file] `reasons/pg.py` — The ~3,536 unseen lines contain every `PgApi` method; enumerate `def ` signatures to know the ported set.
- [function] `reasons/api.py:_pg_dispatch` — The exact dispatch implementation defines the method-signature contract any new `PgApi` method must satisfy.
- [function] `reasons/cli.py:_require_sqlite` — Grep for every call site; this is the authoritative list of still-gated CLI commands.
- [file] `tests/test_pg_dispatch.py` — The remaining 373 lines likely enumerate the PG/ SQLite parity expectations for each command.
- [general] `report_gated-composition` — Determine whether `report_gated` needs its own `PgApi` method or inherits portability from `list_gated`.

## Beliefs

- `pg-dispatch-requires-pgapi-method` — Every API function that calls `_pg_dispatch(name)` requires a `PgApi.<name>` method on `reasons/pg.py:PgApi`, because the dispatch layer resolves the method by name via `getattr`.
- `list-gated-tests-pg-dispatch` — `list_gated` is already ported: `reasons/api.py:3639` dispatches the command to `PgApi` by name.
- `require-sqlite-prevents-pg-execution` — `_require_sqlite` in `reasons/cli.py` raises `SystemExit` when `--pg` or `REASONS_PG_CONNINFO` is set, which is what "still gated" means in the CLI.
- `hash-sources-is-still-gated` — `hash-sources` is one of the 19 gated commands, per `tests/test_pg_dispatch.py::TestRequireSqlite::test_pg_set_exits`.
- `all-pg-tables-scope-by-domain` — Every table in `reasons/pg.py:SCHEMA` carries `domain_id` in its primary key, so any new `PgApi` method must filter on `self.domain_id` to preserve multi-tenant isolation.
- `raw-helpers-do-not-commit` — `_add_node_raw` explicitly documents "Does NOT commit"; public `PgApi` methods (like `init_db`) are responsible for `self.conn.commit()`.
