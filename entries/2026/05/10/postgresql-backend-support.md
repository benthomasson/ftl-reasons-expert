# PostgreSQL Backend Support

**Date:** 2026-05-10
**PR:** [#128](https://github.com/benthomasson/ftl-reasons/pull/128) (merged)
**Issue:** [#127](https://github.com/benthomasson/ftl-reasons/issues/127)
**Version:** 0.39.0

## What Changed

The `reasons` CLI can now operate directly against a PostgreSQL database instead of its default SQLite file. Two new global flags — `--pg <conninfo>` and `--project-id <uuid>` — route every compatible command through the existing `PgApi` class in `reasons_lib/pg.py`, which was previously only reachable through the expert-service HTTP API. Environment variables `REASONS_PG_CONNINFO` and `REASONS_PROJECT_ID` serve as alternatives to avoid exposing connection strings in process listings.

## Architecture

### The Dispatch Layer

A single helper in `api.py` handles all PostgreSQL routing:

```python
def _pg_dispatch(pg_conninfo, project_id, method_name, **kwargs):
    from .pg import PgApi
    with PgApi(pg_conninfo, project_id) as pg:
        return getattr(pg, method_name)(**kwargs)
```

Every PG-compatible API function gained `pg_conninfo=None, project_id=None` parameters. When `pg_conninfo` is set, the function returns early via `_pg_dispatch` instead of opening a SQLite database. This keeps the dispatch pattern uniform: no subclassing, no abstract backends, no factory — just a two-line early return at the top of each function.

On the CLI side, `_backend_kwargs(args)` inspects the `--pg` flag (or env vars) and returns either `{"db_path": args.db}` or `{"pg_conninfo": ..., "project_id": ...}`. Every `cmd_*` function unpacks this with `**_backend_kwargs(args)`.

### What's PG-Compatible vs SQLite-Only

**25 commands dispatch to PostgreSQL**: init, add, add-justification, retract, assert, what-if, status, show, explain, trace, search, list, list-gated, export, export-markdown, compact, log, nogood, find-culprits, challenge, defend, convert-to-premise, remove-justification, update.

**21 commands remain SQLite-only**: summarize, supersede, trace-access-tags, propagate, add-repo, repos, import-agent, sync-agent, import-beliefs, import-json, hash-sources, check-stale, lookup, ask, deduplicate, derive, accept, list-negative, review-beliefs, detect-contradictions, namespaces. These get a `_require_sqlite` guard that prints an error and exits if `--pg` is set.

The SQLite-only commands fall into three categories:

1. **Filesystem-dependent** — `hash-sources`, `check-stale`, `add-repo` work with local file paths that don't exist on a remote database server.
2. **LLM-powered** — `derive`, `review-beliefs`, `detect-contradictions`, `ask` call external LLM subprocesses. These could work with PG in principle but need the full network in memory, which is already what SQLite provides.
3. **Import/sync** — `import-json`, `import-agent`, `sync-agent` do bulk operations that assume SQLite's load-entire-network-modify-save pattern.

### Parameter Filtering

Some PgApi methods accept fewer parameters than their SQLite equivalents. The dispatch handles this by raising `NotImplementedError` for unsupported options:

- `search` with `depth != 1` raises — PgApi's neighbor expansion is fixed at depth 1
- `list_nodes` with `challenged`, `min_depth`, `max_depth`, `not_reviewed_since`, `never_reviewed`, or `by_impact` raises — these filters require in-memory graph traversal
- `add_node` / `add_justification` with `namespace` or `any_mode` raises — multi-agent namespacing is not yet implemented in PgApi

This is better than silently ignoring parameters. Users discover unsupported features at call time, not after checking results.

## New PgApi Methods

Four methods were added to `PgApi` to close coverage gaps:

**`export_network(visible_to=None)`** — Queries all nodes, justifications, and nogoods for the project. Returns the same dict format as the SQLite export. Filters private metadata (keys starting with `_`) and respects `visible_to` access tags. This was the highest-impact addition — it unblocks `export-markdown` over PG, which in turn unblocks any command that needs the full network as a dict.

**`remove_justification(node_id, index)`** — Deletes the Nth justification (by serial ID order) from a node. Validates that the node exists, is not a premise, has enough justifications, and that the removal won't leave a derived node with zero justifications. Recomputes truth and propagates.

**`update_node(node_id, text=None, source=None, source_url=None)`** — Dynamic UPDATE that only sets provided fields. Column names are hardcoded (not interpolated from parameters) to prevent SQL injection.

**`convert_to_premise(node_id)`** — Deletes all justifications for a node and sets its truth to IN. Converts a derived belief into a premise. Propagates truth changes.

## The export_markdown PG Path

`export_markdown` in `api.py` needed special handling because it doesn't just pass through to a PgApi method — it calls `export_network` and then reconstructs a `Network` object to pass to the markdown exporter. The PG path:

1. Calls `export_network` to get the raw dict
2. Creates `Node` and `Justification` objects from the dict
3. Builds a `Network` with these nodes
4. Wires the `dependents` index (which justification antecedent points to which dependent node)
5. Passes the `Network` to `_export(net)` — the same internal function the SQLite path uses

This reconstruction is necessary because the markdown exporter expects a `Network` object with wired dependency graphs, not a flat dict.

## Testing

**18 mock-based dispatch tests** (`test_pg_dispatch.py`):
- `_pg_dispatch` routes to the correct PgApi method with correct kwargs
- `_backend_kwargs` returns SQLite kwargs by default, PG kwargs when `--pg` is set, errors when `--project-id` is missing, reads env vars as fallbacks, and lets CLI args override env vars
- `_require_sqlite` passes without `--pg`, errors with `--pg` flag, errors with `REASONS_PG_CONNINFO` env var
- Export-markdown PG path correctly reconstructs Network objects with node IDs, text, justifications, source info, nogoods, and dependents

**24 PG integration tests** (added to `test_pg.py`, require running PostgreSQL):
- `TestExportNetwork` — 8 tests: empty export, premise-only, justifications, outlist nodes, nogoods, source fields, metadata filtering (strips `_` prefixed keys), visible_to filtering
- `TestRemoveJustification` — 7 tests: remove one of two, last-justification guard, premise guard, invalid index, node not found, truth value change, propagation cascade
- `TestUpdateNode` — 4 tests: text update, source+url update, no-op update, not-found error
- `TestConvertToPremise` — 5 tests: derived-to-premise conversion, OUT-to-IN flip, propagation to dependents, not-found error, premise no-op

Total test suite after this change: 855 passed, 131 skipped.

## Design Decisions

**No abstract backend class.** The dispatch is a function-level pattern, not an inheritance hierarchy. Each API function decides whether to delegate to PG or proceed with SQLite. This keeps the change minimal — no new base classes, no refactoring of existing code, no risk of breaking the SQLite path.

**Lazy import of PgApi.** The `from .pg import PgApi` inside `_pg_dispatch` means psycopg is never imported unless `--pg` is actually used. Users who don't need PostgreSQL don't need to install psycopg.

**Fail-loud for unsupported features.** When a PG user passes `--depth 3` to search or `--namespace` to add, they get a `NotImplementedError` with a clear message. No silent degradation.

**Per-operation transactions.** PgApi executes each method as a SQL transaction, unlike SQLite which loads the entire network, modifies it in memory, and saves. This means PgApi supports concurrent writers and multi-tenant deployment out of the box, but operations that need a global view (like `propagate` or `deduplicate`) remain SQLite-only.

## What's Still SQLite-Only

The biggest gap is the LLM-powered commands: `derive`, `review-beliefs`, `detect-contradictions`, `list-negative`. These pull the full network into memory and send portions of it to an LLM. They could work with PG by calling `export_network` first, but this would be a different pattern (load everything, process in memory) rather than PgApi's per-operation transaction model.

The namespace/multi-agent system (`import-agent`, `sync-agent`, `namespaces`) is also SQLite-only. PgApi has multi-tenancy via `project_id`, but namespace-prefixed belief isolation within a single project isn't implemented.

Two gated beliefs in the expert repo track this coverage gap:
- `storage-is-fully-production-grade-across-backends` — gated on `pgapi-partial-api-coverage`
- `verified-correctness-extends-to-all-backends` — gated on `pgapi-partial-api-coverage`

These will flip IN when PgApi reaches full API parity, which may never be a goal — the per-operation transaction model is fundamentally different from the load-modify-save model, and some operations only make sense in one context.
