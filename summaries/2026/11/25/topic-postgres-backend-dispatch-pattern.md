---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How most `reasons/api.py` functions dispatch to `PgApi` via `_pg_dispatch`, and why `report_belief` deviates by routing PG through sub-calls only.

## How the standard `reasons/api.py` → `PgApi` dispatch works

`reasons/api.py` is written as a backend-agnostic facade: public API functions take optional `pg_conninfo` / `domain_id` arguments, and when `pg_conninfo` is provided they hand the operation to PostgreSQL instead of using the SQLite/`Storage` path.

The central helper is:

- `reasons/api.py:184` — `_pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs)`

Its implementation is intentionally small:

```python
def _pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs):
    from .pg import PgApi
    with PgApi(pg_conninfo, domain_id) as pg:
        return getattr(pg, method_name)(**kwargs)
```

So the standard pattern is:

1. Public function is called with `pg_conninfo` and `domain_id`.
2. It immediately branches to `_pg_dispatch(...)`.
3. `_pg_dispatch` lazily imports `PgApi` from `reasons/pg.py:98`.
4. It opens `PgApi` as a context manager.
5. It calls the named method on `PgApi`, forwarding all keyword arguments.
6. It returns that method’s result directly.

Examples visible in `api_file_head`:

- `init_db(...)` dispatches to `_pg_dispatch(..., "init_db", project_name=project_name)`.
- `ensure_namespace(...)` dispatches to `_pg_dispatch(..., "ensure_namespace", namespace=namespace)`.
- `list_namespaces(...)` begins its `pg_conninfo` branch the same way.

The tests in `tests/test_pg_dispatch.py::TestPgDispatch` confirm the contract:

- `_pg_dispatch("postgresql://...", "proj-1", "get_status")` calls `PgApi.get_status()`.
- `_pg_dispatch(..., "show_node", node_id="a")` forwards `node_id="a"` to `PgApi.show_node(...)`.

So for most API functions, PG dispatch is a one-to-one façade: the API function and `PgApi` method share the same operation name, and `PgApi` owns the entire backend operation.

## Why `report_belief` deviates

There are two different `report_belief` definitions in the observations:

- `reasons/api.py:3732` — the public API-level `report_belief`.
- `reasons/report_belief.py:179` — the actual report renderer.

The second one is clearly database-agnostic. `reasons/report_belief.py:report_belief` accepts already-fetched data:

```python
def report_belief(node_id, node_detail, explain_steps, premises_data,
                  source_chunks, model="", timeout=300):
```

It then either:

- builds a structured markdown report with `_build_structured_report(...)`, or
- formats the data and calls `generate_report(...)` when `model` is set.

`reasons/report_belief.py` contains no PostgreSQL connection handling, no `PgApi` import, and no storage access. It is a rendering module: it turns belief data into markdown, optionally using an LLM through `generate_report()`.

That split explains why `reasons/api.py:report_belief` should not dispatch the whole operation as a single `_pg_dispatch(..., "report_belief")` call:

- Report generation is not a primitive database operation.
- It needs several pieces of data: node details, explanation steps, root premises, source chunks.
- Those reads can be performed through lower-level PG/PgApi calls.
- The final rendering should remain shared between backends and live in `reasons/report_belief.py`.
- Putting report rendering and LLM invocation inside `PgApi` would mix persistence with presentation/LLM concerns.

So the likely architecture is:

```text
api.report_belief(... pg_conninfo=...)
    -> fetch node data via PG sub-calls
    -> fetch explanation/premises/source-chunk data via PG sub-calls
    -> call reasons.report_belief.report_belief(...)
    -> return markdown
```

The “sub-calls only” phrasing means the API-level function composes multiple smaller operations instead of dispatching one `PgApi.report_belief(...)` call. Those sub-calls may be direct `_pg_dispatch(...)` calls or calls to sibling API helpers that themselves dispatch. The observations do not show the body of `reasons/api.py:3732`, so the exact sub-call names cannot be confirmed here.

A related pattern appears in the tests for `export_markdown`: `tests/test_pg_dispatch.py::TestExportMarkdownPgPath` patches `reasons.api.export_network` and then lets `export_markdown` format the returned data locally. That is the same architectural shape: PG provides data, Python-side rendering produces the final output.

## What is missing from the observations

The zero match counts for `_pg_dispatch` and `report_belief` references are not reliable evidence that no calls exist. The observations include only:

- `reasons/api.py:184` for `_pg_dispatch`.
- `reasons/api.py:3732` for the API-level `report_belief` definition.
- No body for `reasons/api.py:report_belief`.
- No full method list for `PgApi`.

To name the exact PG sub-calls used by `report_belief`, you would need the body of `reasons/api.py` starting at line 3732 and the relevant `PgApi` methods in `reasons/pg.py`.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the body around line 3732 to see the exact sub-calls used by `report_belief`.
- [file] `reasons/pg.py` — Review `PgApi` methods and confirm whether there is no single `PgApi.report_belief` method.
- [function] `reasons/report_belief.py:report_belief` — Trace the structured-vs-LLM rendering path and verify it is fully database-agnostic.
- [function] `reasons/api.py:_pg_dispatch` — Understand the lazy import, context-manager lifetime, and keyword-forwarding contract.
- [file] `tests/test_pg_dispatch.py` — Compare dispatch tests with composite PG paths such as `export_markdown`.

## Beliefs

- `_pg-dispatch-is-pgapi-facade` — `_pg_dispatch` at `reasons/api.py:184` opens `PgApi(pg_conninfo, domain_id)` and returns `getattr(pg, method_name)(**kwargs)`.
- `pg-dispatch-forwards-kwargs` — The tests in `tests/test_pg_dispatch.py::TestPgDispatch` verify that keyword arguments are forwarded to the named `PgApi` method.
- `report-belief-is-two-layer` — `report_belief` exists in both `reasons/api.py:3732` as an API facade and `reasons/report_belief.py:179` as a renderer.
- `report-belief-renderer-uses-prefetched-data` — `reasons/report_belief.py:report_belief` consumes `node_detail`, `explain_steps`, `premises_data`, and `source_chunks` rather than querying storage itself.
- `report-belief-renderer-selects-llm-or-structured` — In `reasons/report_belief.py:179`, a non-empty `model` invokes `generate_report`; otherwise it calls `_build_structured_report`.
