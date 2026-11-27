---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Understand which commands support PostgreSQL and why this command explicitly rejects it.

## Which commands support PostgreSQL — and why some explicitly reject it

### The dispatch architecture

PostgreSQL support is opt-in through a per-call `pg_conninfo` argument (plus `domain_id`) supplied by `_backend_kwargs(args)` in `reasons/cli.py` (see `tests/test_pg_dispatch.py:TestBackendKwargs`). Every public API function follows the same shape:

1. If `pg_conninfo` is set, delegate to `_pg_dispatch` (`reasons/api.py:184-188`), which opens a `PgApi` context (`with PgApi(pg_conninfo, domain_id) as pg`) and calls `getattr(pg, method_name)(**kwargs)`.
2. Otherwise fall through to the SQLite code path (`Storage` in `reasons/storage.py`).

`PgApi` lives in `reasons/pg.py` and is a *transactional, no-full-network-load* backend — one SQL transaction per operation. Its docstring (`pg.py:1-6`) is the design contract: "no in-memory Network object."

### Commands that DO support PostgreSQL

The observation set shows `_pg_dispatch` calls for at least:

| Command / API fn | api.py lines | PgApi method |
|---|---|---|
| `init-db` | 203-204, 226-227 | `init_db`, `ensure_namespace` |
| `list-namespaces` | 251-252 | `list_namespaces` |
| `add` (node) | 315 | `add_node` |
| `add-justification` | 434 | `add_justification` |
| `retract` | 485 | `retract_node` |
| `what-if retract` | 524 | `what_if_retract` |
| proposals (`reject` shown) | 5811 | `reject_proposal` |

`tests/test_pg_dispatch.py` confirms additional PG-dispatchable commands: `export_markdown`, `import_json`, `import_beliefs`, `import_agent`, `sync_agent`, `hash_sources`, `check_stale`, `lookup`, `add_repo`, `list_repos`, `list_negative`. `tests/test_pg.py` additionally validates `get_status`, `show_node`, `search`, `list_nodes`, `list_gated`, `assert_node` — so the CLI commands backed by those all work against PostgreSQL.

### Commands that explicitly reject PostgreSQL

There are two distinct rejection mechanisms:

**1. CLI-level block via `_require_sqlite`.** Several commands exist only in the SQLite code path. In `reasons/cli.py`:

- `cmd_mark_duplicate` (line 157) — "mark-duplicate is not supported with --pg (no PgApi implementation)"
- `cmd_mark_superseded` (line 179) — same pattern
- `cmd_defeat_justification` (line 201)
- `cmd_defeat_with_scope` (line 232)
- `cmd_migrate_defeaters` (line 287)
- `cmd_classify_defeaters` (line 324)
- Generic fallback at line 1305: `f"Error: {command_name} is not supported with --pg"` — this is the shared `_require_sqlite(args, command_name)` helper exercised by `tests/test_pg_dispatch.py:TestRequireSqlite`.

**2. API-level `NotImplementedError`** raised when `pg_conninfo` is present but a feature can't be honored:

- `any_mode` on `add_node` (api.py:314) and `add_justification` (api.py:433) — "any_mode is not supported with PostgreSQL"
- `add_tags` (942), `remove_tags` (966), `get_tags` (993)
- `add_source` (1013)
- `depth` (3053) and `tag filtering` (3055) in search/traverse
- An aggregator at api.py:3437 that raises `NotImplementedError(f"{', '.join(unsupported)} not supported with PostgreSQL")`

### Why they reject it

The rejections fall into three categories:

**A. No `PgApi` implementation exists.** `PgApi.__init__` requires `psycopg` (see `_require_psycopg` in pg.py). If the SQL path doesn't have a matching method on `PgApi`, dispatch would fail with `AttributeError`. Rather than leave that failure mode implicit, CLI commands check up front via `_require_sqlite` and exit(1) with a clear message. This is why `mark-duplicate`, `mark-superseded`, `defeat-*`, `migrate-defeaters`, and `classify-defeaters` are blocked — `PgApi` explicitly "has no implementation" for these (line 157 comment is the smoking gun).

**B. Schema gap — features that need tables not in the PG schema.** The PG schema (`reasons/pg.py:22-97`) defines `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, `rms_network_meta`. It does **not** define `node_tags` or `node_sources` tables (contrast with SQLite's `SCHEMA` in `reasons/storage.py:63-83`). In the PG path, access tags are persisted inside `rms_nodes.metadata` JSONB (see `_add_node_raw` in pg.py:177-190 and `metadata["access_tags"]` in `add_node`). So `add-tags` / `remove-tags` / `get-tags` / `add-source` would require either a new join table or JSONB querying — neither of which exists yet. The message is `"not supported with PostgreSQL yet"` (note the *yet*).

**C. Algorithmic/semantic gap — features that are hard to express without in-memory state.** These are the interesting "why" cases:

- **`any_mode`** (api.py:314, 433). This flag expands a single SL with N antecedents into N separate justifications (one per antecedent, i.e., OR semantics). In the SQLite path this is done during justification parsing *before* insertion. The `_pg_dispatch(...)` call at api.py:315/434 is missing the `any_mode` keyword, and the `if pg_conninfo:` guard at 312-314 / 431-433 intercepts first. The expansion is a *pre-processing* step that lives in the SQLite add flow; it simply wasn't ported to the transactional path. It's a mechanical fix, but the current code deliberately refuses rather than silently producing AND semantics that contradict what the user asked for.

- **`depth`** (api.py:3053) and **tag filtering** (3055). PG's design tenant is "no full-network load, each op is a transaction." Computing a hop-depth requires a recursive traversal (a recursive CTE) which has not been written, and tag filtering requires either the missing `node_tags` table or a JSONB containment query. Both are consequences of the SQL backend choosing a normalized/tabular model over the in-memory `Network` graph.

**Summary of intent:** the rejection is *explicit and deliberate*. Rather than silently degrade — e.g., running `any_mode` as AND, filtering on the wrong column, or crashing with `AttributeError` inside `_pg_dispatch` — API and CLI layers raise `NotImplementedError` or call `sys.exit(1)` with a "no PgApi implementation" or "not supported with PostgreSQL yet" message. The recurring word *yet* on the API-side tags/sources errors (942, 966, 993, 1013) signals these are deferred implementations, not permanent exclusions.

### What's ambiguous

The prompt says "**this** command" (singular) but the observations reveal ~19 distinct rejection messages. If a specific command was intended, the observation set is missing the context to single it out. Given the evidence, the most systematic example is the `_require_sqlite` gate that blocks `mark-duplicate`, `mark-superseded`, `defeat-justification`, `defeat-with-scope`, `migrate-defeaters`, and `classify-defeaters`, plus the `any_mode` guard on `add` / `add-justification`.

## Topics to Explore

- [function] `reasons/api.py:_pg_dispatch` — The single entry point every PG-supporting API function funnels through; adding a new command's PG path starts here.
- [file] `reasons/pg.py` — The `PgApi` class body is the authoritative list of what PostgreSQL can do; features not represented as methods are exactly the ones that get rejected.
- [function] `reasons/cli.py:_require_sqlite` — The helper (referenced by `tests/test_pg_dispatch.py:TestRequireSqlite`) that exits when `--pg` or `REASONS_PG_CONNINFO` is set for commands without a PgApi method.
- [general] `any-mode-port-to-pg` — Understand how `any_mode` expands a single SL into per-antecedent justifications in the SQLite path, since porting it to `PgApi.add_node`/`add_justification` is likely the smallest gap to close.
- [file] `tests/test_pg.py` — Behavioral expectations for the PG backend; also shows which operations are considered "supported" via `pg_api` fixtures.

## Beliefs

- `pg-dispatch-requires-conninfo-and-domain-id` — `_pg_dispatch` and `_backend_kwargs` require both `pg_conninfo` and `domain_id`; missing `domain_id` triggers a `SystemExit` (per `TestBackendKwargs.test_pg_missing_domain_id_exits`).
- `pg-backend-avoids-full-network-load` — `PgApi` performs one SQL transaction per operation and never materializes an in-memory `Network`, per its module docstring in `reasons/pg.py`.
- `pg-access-tags-live-in-node-metadata-jsonb` — Access tags are stored under `metadata["access_tags"]` inside `rms_nodes.metadata`, not in a dedicated join table, which is why `get_tags`/`add_tags`/`remove_tags` raise `NotImplementedError`.
- `cli-rejects-unimplemented-commands-via-systemexit` — `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, and `cmd_classify_defeaters` call `sys.exit(1)` when `--pg` or `REASONS_PG_CONNINFO` is set.
- `any-mode-rejection-prevents-silent-and-semantics` — `api.add_node` and `api.add_justification` raise `NotImplementedError` before dispatch when `any_mode=True` and `pg_conninfo` is set, ensuring OR-expansion is never silently degraded to AND semantics.
