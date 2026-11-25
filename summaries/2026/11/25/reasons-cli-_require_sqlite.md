---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: _require_sqlite in reasons/cli.py

## `_require_sqlite` — Guard that rejects PostgreSQL mode

### Location
`reasons/cli.py` (module-level helper, defined right after `_backend_kwargs`).

### 1. Purpose
It is a **backend compatibility guard**. A handful of CLI commands (`check-integrity`, `backfill-hashes`, `pin-sources`, `pin-update`, `pin-lines`, `ask`, `cluster-list`, `deduplicate`, `derive`, `accept`, `build-wiki`, `review-beliefs`, `review-justifications`, `review-premises`, `repair-premises`, `propose-update`, `repair-smuggled`, `repair`, `contradictions`) are only implemented against the SQLite backend. The function detects whether the user selected PostgreSQL (via `--pg` on the command line or via the `REASONS_PG_CONNINFO` environment variable) and, if so, prints a user-facing error and exits with status 1. In SQLite mode it is a no-op that lets the command continue.

It is essentially the extracted common form of the inline check `if getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO"):` that appears in the older `mark-duplicate`, `mark-superseded`, `defeat-justification`, `defeat-with-scope`, `migrate-defeaters`, and `classify-defeaters` handlers.

### 2. Contract
- **Precondition**: Called at the top of a CLI command handler, *before* any `api.*` call that assumes a SQLite database file.
- **Postcondition on return**: `None` is returned and the caller can safely continue assuming SQLite mode.
- **Postcondition on failure**: The process has already terminated (`SystemExit(1)`). Control never returns to the caller when PG mode is active.
- **Invariant**: Detection is identical to the one in `_backend_kwargs` (`getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO")`), so a command guarded by `_require_sqlite` will never *also* be routed to PostgreSQL by `_backend_kwargs`.
- **Important semantic caveat**: it does *not* verify that SQLite is available or that `args.db` exists. It only forbids PG. The name is slightly misleading — it's a "reject PG" check, not a "require SQLite" check.

### 3. Parameters
- `args` (`argparse.Namespace`): the parsed CLI arguments. Only `args.pg` is read, and even that is defensive via `getattr(args, "pg", None)` so a subparser that forgot to define `--pg` still works (it just falls through to the env-var check). Empty string `--pg ""` is falsy and therefore treated as "no PG".
- `command_name` (`str`): a human-readable label included in the error message. It is **not** necessarily the subparser name — e.g. `cmd_contradictions` passes `"detect-contradictions"` even though the actual CLI subcommand is `"contradictions"`. This is a display string; nothing validates it.

### 4. Return Value
Returns `None` in the SQLite/no-PG path. It never returns in the PG path because it calls `sys.exit(1)` (raises `SystemExit(1)`). Callers therefore have no conditional handling to do — they simply write the guard at the top of the handler and then proceed with SQLite-only logic.

### 5. Algorithm
1. `pg = getattr(args, "pg", None)` — pull the flag, tolerating a missing attribute.
2. If that is falsy, evaluate `os.environ.get("REASONS_PG_CONNINFO")` — which yields `None` when unset, and possibly the empty string when set but blank.
3. `or` short-circuits: pick whichever source is truthy first. Empty strings from either side are ignored, matching the semantics used in `_backend_kwargs`.
4. If `pg` is truthy: `print("Error: <command_name> is not supported with --pg", file=sys.stderr)` then `sys.exit(1)`.
5. Otherwise, the function falls off the end and returns `None`.

### 6. Side Effects
- **Writes to stderr**: exactly one line, of the form `Error: <command_name> is not supported with --pg`.
- **Terminates the process**: `sys.exit(1)` raises `SystemExit` with exit code 1. In a shell this is an error exit; in tests it must be caught (`pytest.raises(SystemExit)` or capsys assertions on stderr).
- **No mutations** to `args`, `os.environ`, or any global state.

### 7. Error Handling
- No exceptions are raised from `getattr` or `os.environ.get` (both are total functions for these argument shapes).
- The only "error path" is `sys.exit(1)`, which is a cooperative exit, not a caught exception. Callers therefore cannot recover; if you need recoverable behavior, this helper is the wrong tool.
- If `command_name` is empty, the message still prints but is not useful — nothing enforces non-empty.

### 8. Usage Patterns
Typical call at the very top of a handler:
```python
def cmd_derive(args):
    _require_sqlite(args, "derive")
    ...   # all code below assumes SQLite
```
Callers must place it before any code path that could touch `args.db`, `api.*`, or the environment — otherwise the guard is moot. The helper is also assumed to run before `_backend_kwargs(args)`, since `_backend_kwargs` is what would otherwise produce `{"pg_conninfo": ..., "domain_id": ...}`.

A few older handlers (`cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, `cmd_classify_defeaters`) still inline the same check rather than calling this helper — an inconsistency worth knowing about if you tabulate which commands are PG-safe.

### 9. Dependencies
- `os` (stdlib) — `os.environ.get`.
- `sys` (stdlib) — `sys.stderr`, `sys.exit`.
- `argparse` only implicitly, via the `args` object shape (`args.pg`).
- No import of `reasons.api`; the helper is intentionally decoupled from the API dispatch so it can guard commands before any backend is constructed.

### Assumptions not enforced by the type system
- That `args` is an `argparse.Namespace` (duck-typed; any object with an optional `pg` attribute works).
- That `REASONS_PG_CONNINFO` and `--pg` are semantically interchangeable — the rest of the codebase (e.g. `_backend_kwargs`) relies on this same equivalence, but there is no validation that a `--pg` string is a valid libpq conninfo.
- That exiting with `SystemExit(1)` is the correct UX for a CLI — this would be wrong in library use.
- That `command_name` matches what the user typed; in the `contradictions` case it does not.

## Topics to Explore
- [function] `reasons/cli.py:_backend_kwargs` — Companion helper that produces the backend kwargs dict; uses the same PG-detection expression and would route to PG if it were not preempted by `_require_sqlite`.
- [file] `reasons/api.py` — Determines which `api.*` functions actually support a PgApi path (the ones that don't are the candidates for `_require_sqlite` guards).
- [file] `tests/test_pg_dispatch.py` — Tests that verify CLI-to-backend dispatch, including which commands are rejected under PG.
- [file] `tests/test_cli.py` — Behavioral CLI tests; look for `pytest.raises(SystemExit)` or capsys assertions around `--pg`.
- [general] `pg-dispatch-inconsistency` — Several handlers (`mark-duplicate`, `defeat-*`, `migrate-defeaters`, `classify-defeaters`) still inline the same check instead of calling `_require_sqlite`; a cleanup opportunity if these should ever be unified.

## Beliefs
- `require-sqlite-exits-on-pg` — When `--pg` or `REASONS_PG_CONNINFO` is truthy, `_require_sqlite` writes an error to stderr and raises `SystemExit(1)`.
- `require-sqlite-returns-none-when-sqlite` — When neither PG source is set, `_require_sqlite` returns `None` and the command continues with SQLite.
- `pg-detection-shared-between-helpers` — `_require_sqlite` and `_backend_kwargs` both compute PG mode with `getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO")`.
- `pg-guard-not-uniformly-applied` — `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, and `cmd_classify_defeaters` inline the PG check rather than calling `_require_sqlite`.
- `command-name-can-diverge-from-cli-name` — `cmd_contradictions` passes the label `"detect-contradictions"` to `_require_sqlite`, even though the CLI subcommand is `contradictions`.
