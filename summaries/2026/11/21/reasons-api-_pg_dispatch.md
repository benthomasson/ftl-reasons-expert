---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _pg_dispatch in reasons/api.py

# `_pg_dispatch` — Backend Routing Shim

## Purpose

`_pg_dispatch` is the single, centralized routing point that lets the entire functional API in `reasons/api.py` be backed by **two different storage engines** — a local SQLite `Storage` and a remote PostgreSQL `PgApi` — without duplicating the "if Postgres, do X" logic in every API function.

Every public function in `api.py` that supports Postgres uses the same idiom at the top:

```python
def some_operation(..., pg_conninfo=None, domain_id=None):
    if pg_conninfo:
        return _pg_dispatch(pg_conninfo, domain_id, "some_operation", ...)
    # ... SQLite implementation ...
```

`_pg_dispatch` is therefore the "escape hatch" out of the SQLite-only code path. It exists to:
1. Hide the `PgApi` import so that callers who never use Postgres don't pay the import cost (and don't need `psycopg` installed).
2. Guarantee every dispatched call opens and closes its own `PgApi` connection (no long-lived connections held by the module).
3. Provide a single place to hang the convention that `method_name` corresponds to the public function name on `PgApi`.

## Contract

- **Precondition (enforced by callers, not this function):** `pg_conninfo` is truthy. Every call site is gated by `if pg_conninfo:`. The function itself never checks — passing `None`/`""` will just get forwarded to `PgApi`.
- **Precondition (not type-enforced):** `method_name` is a name of a callable attribute on `PgApi` (or inherited from its bases).
- **Precondition (not type-enforced):** `**kwargs` match the `PgApi.<method_name>` signature — usually *not* the sqlite signature. Some callers drop arguments that Postgres doesn't accept (e.g., `init_db` drops `force` and `db_path`).
- **Postcondition:** The result of `PgApi.<method_name>(**kwargs)` is returned unchanged — no wrapping, no dict normalization.
- **Invariant:** The `PgApi` instance is entered and exited exactly once per call, i.e., a fresh connection per dispatch. Callers should not assume a connection persists between two dispatches.

## Parameters

| Name | Type (implicit) | Meaning |
|---|---|---|
| `pg_conninfo` | PostgreSQL DSN / connection info (any value `PgApi.__init__` accepts) | Passed straight through. Could be a URL string, a `psycopg.connect`-compatible string, etc. The type is not annotated, so the contract is "whatever `PgApi` accepts". |
| `domain_id` | Domain / tenant identifier | Passed through to `PgApi`. No type annotation, no validation here. |
| `method_name` | `str` | The name of the method to call on the `PgApi` instance. Looked up via `getattr`. |
| `**kwargs` | Keyword args | Forwarded verbatim to the target method. **A mismatch is a `TypeError` at call time, not a compile-time error.** |

Edge cases to be aware of:
- `pg_conninfo` being falsy will not be rejected here — the current `PgApi` may connect to localhost or fail. Callers are expected to guard.
- `domain_id=None` is legal at the type level but may break at call time depending on the `PgApi` implementation.
- `method_name` is a bare string. There is no whitelist; a typo (`"add_ndoe"`) becomes an `AttributeError`, not a silent no-op.

## Return Value

Whatever `PgApi.<method_name>` returns. In practice this is (per the module docstring) a dict suitable for JSON serialization, but the function performs **no validation, conversion, or defaulting**. If the remote method returns `None`, that `None` propagates to the caller. Callers are obligated to handle whatever shape the corresponding `PgApi` method documents — and to keep those shapes in sync with the SQLite branch.

## Algorithm

1. **Lazy import** of `PgApi` from `.pg`. This is inside the function body so importing `reasons.api` never requires `psycopg` to be installed, and avoids a top-level circular import between `api` and `pg`.
2. **Enter context manager:** `with PgApi(pg_conninfo, domain_id) as pg:`. Whatever `PgApi.__init__` / `__enter__` does (connect, begin transaction, set search_path by domain) happens here.
3. **Attribute lookup:** `getattr(pg, method_name)` — dynamic dispatch by name.
4. **Invoke** with `(**kwargs)`, return its value.
5. **Exit context manager:** `PgApi.__exit__` runs, which in a well-implemented `PgApi` will commit on success or rollback on exception, and close the connection.

There is **no** retry, **no** exception translation, **no** fallback to SQLite. It's a pure passthrough with connection lifecycle management.

## Side Effects

- **Network I/O:** Opens a fresh PostgreSQL connection for every call.
- **Database mutation:** Any write method (e.g., `add_node`, `retract_node`, `accept_proposal`) mutates the remote Postgres DB. Commit/rollback semantics live in `PgApi`'s `__enter__`/`__exit__`.
- **Resource lifecycle:** Connection (and possibly a transaction) per invocation. There's no pooling at this level — if the underlying `PgApi` uses a pool internally, that's an implementation detail of `PgApi`.
- **No local state:** This function does not touch `_network_cache`, does not write to SQLite, does not log.

## Error Handling

`_pg_dispatch` catches nothing. Every failure mode is a straight propagation:

| Source | Exception |
|---|---|
| `from .pg import PgApi` fails (psycopg missing) | `ImportError` |
| `PgApi(...)` / `pg.__enter__()` fails (bad DSN, network) | Whatever `PgApi` raises |
| `method_name` not found on `PgApi` | `AttributeError` |
| `kwargs` don't match the method signature | `TypeError` |
| Method raises (e.g., `KeyError` for a missing node, `ValueError` for invalid state, `NotImplementedError` for unsupported args) | That exception propagates |
| `__exit__` raises | Propagates |

**Design consequence:** callers get the *real* backend error, not a wrapped or translated one. This is a deliberate choice — it preserves stack traces and message fidelity, but it means the SQLite branch and Postgres branch may raise *different* exception types for the same conceptual failure. For example, `show_node` raises `KeyError("Node 'x' not found")` in SQLite; the Postgres method is expected to do the same, but that is a social contract, not a compiler-enforced one.

Contrast with sibling functions like `add_tags` and `add_source`, which raise `NotImplementedError` unconditionally when `pg_conninfo` is set — i.e., not every function routes through `_pg_dispatch`. The presence of a `_pg_dispatch` call is the marker of "Postgres is supported for this operation".

## Usage Patterns

The typical shape appears roughly 30 times in `api.py`:

```python
def init_db(db_path=DEFAULT_DB, force=False, project_name="",
            pg_conninfo=None, domain_id=None):
    if pg_conninfo:
        return _pg_dispatch(pg_conninfo, domain_id, "init_db",
                            project_name=project_name)   # note: drops force, db_path
    # ... SQLite path ...
```

Important caller obligations:

1. **Guard on `pg_conninfo` first.** The dispatcher does not check; the caller is the gatekeeper.
2. **Filter kwargs for the Postgres signature.** Notable examples:
   - `init_db` drops `force` and `db_path` (meaningless in Postgres).
   - `import_json` parses the file into JSON and passes `data=data`, so the Postgres method receives the *parsed network dict*, not a path.
   - `import_beliefs` / `import_agent` / `sync_agent` read the files locally (using `_normalize_json` / `_normalize_markdown` when Postgres is in play) and pass strings, not paths.
3. **Reject unsupported features before dispatching.** `add_node` and `add_justification` raise `NotImplementedError` for `any_mode` before dispatching; `search` raises for `depth != 1` and `tag=...`; `list_nodes` raises for `challenged`, `by_impact`, `tag`, etc.
4. **Keep method names aligned with public function names** — this is a naming convention that `tests/test_pg_dispatch.py` presumably verifies.

## Dependencies

- **Internal:** `.pg.PgApi` (imported lazily). Requires `PgApi` to be a context manager.
- **Builtins:** `getattr`.
- **Nothing else.** No logging, no exception handling, no `Storage`, no `Network`.

Notably absent dependencies: no import of `psycopg`, no import of `sqlite3`, no reference to `_with_network` — this function is orthogonal to the SQLite code path.

## Assumptions Not Enforced by the Type System

- `method_name` is a callable attribute of `PgApi` (not a data attribute, not `None`).
- `kwargs` names exactly match the method signature — a caller typing `_pg_dispatch(..., "add_node", nodeid=...)` fails only at runtime.
- `PgApi` returns shapes compatible with the SQLite branch's return value (parallel-implementation contract).
- `pg_conninfo` and `domain_id` are non-`None` / non-empty when dispatch happens.
- `PgApi.__exit__` implements correct commit/rollback semantics — this function has no safety net, so transaction correctness is entirely up to `PgApi`.
- Domain scoping (`domain_id`) is enforced server-side; the dispatcher does no authorization.

## Topics to Explore

- [file] `reasons/pg.py` — Contains `PgApi`; this is where connection setup, domain scoping, and transaction commit/rollback (`__enter__`/`__exit__`) live.
- [function] `reasons/pg.py:PgApi.__exit__` — Determines whether a dispatched write commits or rolls back; essential for understanding the real side-effect story.
- [file] `tests/test_pg_dispatch.py` — Shows which `method_name` values are expected, and likely provides mocked `PgApi` behavior to assert routing.
- [general] `backend-parity` — The social contract that SQLite and Postgres methods return equivalent dict shapes; worth auditing whether `PgApi` methods actually mirror the SQLite branches for e.g. `show_node`, `summarize`, `import_json`.
- [function] `reasons/api.py:add_node` — Best concrete example of the pattern: guards on `any_mode`, filters `accepted_pr`/`source_type`/`db_path` out of the dispatched kwargs, and demonstrates the "caller responsibility" split.

## Beliefs

- `pg-dispatch-requires-truthy-pg-conninfo` — `_pg_dispatch` does not validate `pg_conninfo`; callers must guard with `if pg_conninfo:` before calling it.
- `pg-dispatch-lazy-imports-pgapi` — `PgApi` is imported inside the function body, so `import reasons.api` never requires `psycopg` to be installed.
- `pg-dispatch-opens-and-closes-connection-per-call` — Every dispatch enters a fresh `PgApi` context manager; no connection is retained between dispatches.
- `pg-dispatch-kwargs-must-match-pgapi-signature` — Keyword arguments are forwarded verbatim via `**kwargs`; a mismatch raises `TypeError` at call time, not at import time.
- `pg-dispatch-propagates-exceptions-unwrapped` — The function contains no `try`/`except`; every backend, lookup, or signature error is raised directly to the caller.
