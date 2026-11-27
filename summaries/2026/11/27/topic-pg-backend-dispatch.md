---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How `pg_conninfo`/`domain_id` flow through `_pg_dispatch` and which API functions support PostgreSQL versus raise `NotImplementedError`.

## How `pg_conninfo` / `domain_id` flow

The PostgreSQL path is opt-in: a caller passes `pg_conninfo` and `domain_id` into a public API function. If `pg_conninfo` is truthy, the function delegates to PostgreSQL via `_pg_dispatch`; otherwise it uses the SQLite path.

1. **CLI/backend selection**
   - `tests/test_pg_dispatch.py::TestBackendKwargs` shows `_backend_kwargs` in `reasons/cli.py` returns either `{"db_path": ...}` for SQLite or `{"pg_conninfo": ..., "domain_id": ...}` for PostgreSQL.
   - Environment fallbacks are covered: `REASONS_PG_CONNINFO` and `REASONS_PROJECT_ID`.
   - The CLI enforces that `--pg` has a `domain_id`; `TestBackendKwargs.test_pg_missing_domain_id_exits` expects `SystemExit`.
   - Some CLI commands explicitly reject PG via `_require_sqlite`, e.g. `hash-sources` in `TestRequireSqlite`.

2. **Public API signatures**
   - Functions such as `add_node` at `reasons/api.py:293`, `add_justification` at `:415`, `retract_node` at `:474`, `get_status` at `:788`, and `show_node` at `:871` accept:
     ```python
     pg_conninfo=None, domain_id=None
     ```
   - They check `if pg_conninfo:` before dispatching. Example: `add_node` checks at `reasons/api.py:312` and calls `_pg_dispatch(..., "add_node", ...)` at `:315`.

3. **Central dispatch**
   - `_pg_dispatch` is defined at `reasons/api.py:184`:
     ```python
     def _pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs):
     ```
   - It opens a PostgreSQL API object at `reasons/api.py:187`:
     ```python
     with PgApi(pg_conninfo, domain_id) as pg:
     ```
   - It then calls the requested `PgApi` method with `**kwargs`. Tests confirm this:
     - `TestPgDispatch.test_dispatch_calls_pgapi_method`
     - `TestPgDispatch.test_dispatch_passes_kwargs`

4. **`PgApi` receives and stores `domain_id`**
   - `PgApi.__init__` in `reasons/pg.py` accepts `(conninfo, domain_id)`.
   - If `conninfo` is a string, it opens a psycopg connection; otherwise it treats it as an existing connection.
   - It stores `self.domain_id = str(domain_id)`.
   - `PgApi` SQL is domain-scoped, e.g. `rms_nodes` has primary key `(id, domain_id)` and queries filter by `domain_id = %s`.

5. **Context-manager behavior**
   - `PgApi.__exit__` rolls back on exception and closes the connection if `PgApi` owns it.
   - So exceptions raised by PostgreSQL-backed methods roll back the transaction.

## Visible API dispatch sites that support PostgreSQL

Based on the provided matches, these public API functions have explicit PostgreSQL dispatch calls:

| API function / site | Dispatch line | PG method |
|---|---:|---|
| `init_db` area, signature near `reasons/api.py:193` | `:204` | `"init_db"` |
| `ensure_namespace` area, signature near `:217` | `:227` | `"ensure_namespace"` |
| `list_namespaces` area, signature near `:243` | `:252` | `"list_namespaces"` |
| `add_node` at `:293` | `:315` | `"add_node"` |
| `add_justification` at `:415` | `:434` | `"add_justification"` |
| `retract_node` at `:474` | `:485` | `"retract_node"` |
| `what_if_retract` at `:513` | `:524` | `"what_if_retract"` |
| `what_if_assert` at `:581` | `:591` | `"what_if_assert"` |
| `what_if_supersede` at `:649` | `:659` | `"what_if_supersede"` |
| `assert_node` at `:756` | `:762` | `"assert_node"` |
| `propagate` at `:774` | `:776` | `"propagate"` |
| `get_status` at `:788` | `:800` | `"get_status"` |
| `show_node` at `:871` | `:878` | `"show_node"` |

Tests also show PG dispatch for:
- `import_json` → `PgApi.import_json(data=data)`
- `import_beliefs` → `PgApi.import_beliefs(beliefs_text=...)`
- `import_agent` → `PgApi.import_agent(agent_name=..., claims=...)`
- `export_markdown` with PG kwargs delegates through `export_network(...)`

## PG guards that raise `NotImplementedError`

Some functions are partially or fully unsupported on PostgreSQL.

| Location | Message / meaning |
|---|---|
| `reasons/api.py:314` | `any_mode is not supported with PostgreSQL` in `add_node` |
| `reasons/api.py:433` | `any_mode is not supported with PostgreSQL` in `add_justification` |
| `reasons/api.py:942` | `add_tags is not supported with PostgreSQL yet` |
| `reasons/api.py:966` | `remove_tags is not supported with PostgreSQL yet` |
| `reasons/api.py:993` | `get_tags is not supported with PostgreSQL yet` |
| `reasons/api.py:1013` | `add_source is not supported with PostgreSQL yet` |
| `reasons/api.py:3053` | `depth is not supported with PostgreSQL` |
| `reasons/api.py:3055` | `tag filtering is not supported with PostgreSQL` |
| `reasons/api.py:3436` | `NotImplementedError(` with truncated message; enclosing function not shown |

Important nuance: `add_node` and `add_justification` are not wholly unsupported. They dispatch to PostgreSQL for normal usage, but explicitly reject `any_mode` when `pg_conninfo` is set.

## What is missing / caveat

The `find_pg_conninfo_usages` result has `total_count: 236`, but only a subset of matches is shown. Many `def` names are also omitted from the match snippets. Therefore the table above is the visible set, not a guaranteed exhaustive list of every PostgreSQL-capable API function. To enumerate all of them, inspect every `if pg_conninfo:` site and every `NotImplementedError` guard in `reasons/api.py`.

## Topics to Explore

- [file] `reasons/api.py` — Inspect all `if pg_conninfo:` dispatch sites and `NotImplementedError` guards between lines 184–1013 and around 3053/3436.
- [function] `reasons/api.py:_pg_dispatch` — Central PostgreSQL dispatch helper that constructs `PgApi` and invokes the named method.
- [function] `reasons/pg.py:PgApi.__init__` — Shows how `conninfo` is opened and how `domain_id` is normalized and stored.
- [file] `tests/test_pg_dispatch.py` — Tests confirm dispatch kwargs, backend selection, env vars, and PG rejection for some CLI commands.
- [general] `postgresql-domain-isolation` — Verify every `PgApi` query is scoped by `domain_id`, since the schema uses composite keys like `(id, domain_id)`.

## Beliefs

- `pg-dispatch-requires-pg-conninfo` — Public API functions route to PostgreSQL only when `pg_conninfo` is truthy; otherwise they use the SQLite path.
- `pg-dispatch-instantiates-pgapi` — `_pg_dispatch` constructs `PgApi(pg_conninfo, domain_id)` and calls the requested `PgApi` method with `**kwargs`.
- `pgapi-stores-domain-id` — `PgApi.__init__` converts `domain_id` to a string and stores it as `self.domain_id`.
- `pgapi-scopes-sql-by-domain-id` — PostgreSQL tables and queries use `domain_id` for multi-tenant isolation, e.g. `rms_nodes` primary key `(id, domain_id)`.
- `add-node-pg-rejects-any-mode` — `add_node` raises `NotImplementedError` for `any_mode` when using PostgreSQL at `reasons/api.py:314`.
