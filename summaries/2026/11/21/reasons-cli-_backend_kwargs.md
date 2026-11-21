---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _backend_kwargs in reasons/cli.py

## `_backend_kwargs` — Backend Selection Adapter

### Purpose

`_backend_kwargs` is a private helper that translates the parsed CLI arguments (`argparse.Namespace`) into the keyword-argument set a backend-specific API function expects. It resolves two orthogonal decisions:

1. **Which storage backend** to use — SQLite (default) or PostgreSQL.
2. **The connection parameters** for that backend (`db_path` vs. `pg_conninfo` + `domain_id`).

It exists so that every `cmd_*` command function can hand a consistent `**kwargs` dict to `reasons.api.*` without duplicating the CLI/env lookup and validation logic in ~40 handlers. Each CLI subcommand therefore stays backend-agnostic; the actual dispatch to `SqliteApi` vs. `PgApi` happens inside `reasons.api` based on which keys are present in the kwargs.

### Contract

**Preconditions**
- `args` must expose a `db` attribute (accessed directly, no `getattr` fallback). In practice `main()` always sets this via the top-level `--db` flag, but a hand-constructed `Namespace` (e.g., in tests) that omits `db` will `AttributeError` if the pg branch is not taken.
- `args` may or may not have `pg`, `domain_id`, and `project_id` attributes; all are accessed via `getattr(..., None)`.
- Environment variables `REASONS_PG_CONNINFO`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID` may be set.

**Postconditions**
- Returns exactly one of two dict shapes:
  - `{"db_path": <str>}` — when no PostgreSQL connection string is available.
  - `{"pg_conninfo": <str>, "domain_id": <str>}` — when PostgreSQL is selected and a domain ID is resolvable.
- If PostgreSQL is selected but no domain ID can be resolved, the process terminates via `sys.exit(1)` after printing an error. **This is a non-recoverable side effect** — callers must not expect to receive control back on that path.

**Invariants**
- The two returned keyspaces are mutually exclusive; callers cannot reasonably mix `db_path` with `pg_conninfo`.
- When `--pg` / `REASONS_PG_CONNINFO` is set, `args.db` is **silently ignored** — a user who supplies both `--pg` and `--db` gets pg behavior with no warning.

### Parameters

| Parameter | Meaning | Notes |
|---|---|---|
| `args` | The parsed `argparse.Namespace` from `main()` | Needs `db`; other attributes looked up defensively via `getattr`. |

Individual fields consulted:

- `args.pg` (str \| None) — connection string from `--pg`.
- `args.domain_id` (str \| None) — from `--domain-id`.
- `args.project_id` (str \| None) — **deprecated alias** for `--domain-id`. Explicitly accepted here even though `main()` marks it as such in its help text; no deprecation warning is emitted.
- `args.db` (str) — SQLite path.
- Env vars, in fallback order: `REASONS_PG_CONNINFO`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID`.

Edge cases:
- Empty strings (`""`) are falsy and fall through the `or` chain — so `--pg ""` falls back to the env var rather than disabling pg. Likewise for `--domain-id ""`.
- No type coercion is performed; `did` is always whatever `argparse` produced (a string) or whatever env var returned (a string). No integer/`UUID` validation.

### Return Value

```python
{"pg_conninfo": pg, "domain_id": did}   # PostgreSQL selected
# or
{"db_path": args.db}                    # SQLite fallback
```

The caller always splats this into an API call:

```python
api.add_node(..., **_backend_kwargs(args))
```

API functions must therefore accept the **union** of both key sets. Whether the target function actually implements pg is not checked here — several commands (`mark-duplicate`, `mark-superseded`, `check-integrity`, etc.) instead use `_require_sqlite` to reject pg before calling `_backend_kwargs`-style code.

### Algorithm

1. **Resolve pg connection string**: `args.pg or os.environ["REASONS_PG_CONNINFO"]` — CLI wins over env.
2. **Resolve domain ID**: first truthy value among
   - `args.domain_id` (`--domain-id`)
   - `args.project_id` (`--project-id`, deprecated)
   - `REASONS_DOMAIN_ID`
   - `REASONS_PROJECT_ID` (also deprecated)
3. **Branch on pg truthiness**:
   - **If `pg` is truthy**:
     - If `did` is falsy → print `Error: --domain-id is required with --pg` to stderr, then `sys.exit(1)`. Exit code 1.
     - Otherwise return `{"pg_conninfo": pg, "domain_id": did}`.
   - **Else**: return `{"db_path": args.db}` — no validation of `db`.

Note the ordering nuance: because `args.project_id` is checked before `REASONS_DOMAIN_ID`, an explicitly passed deprecated `--project-id` will *override* the modern env var `REASONS_DOMAIN_ID`. Symmetrically, `--domain-id` and `--project-id` beat both env vars.

### Side Effects

- **stderr write**: `print(..., file=sys.stderr)` on the pg-without-domain path.
- **Process termination**: `sys.exit(1)` raises `SystemExit`, which unwinds past all `cmd_*` handlers up to the interpreter. Commands cannot catch this with `try/except Exception` (SystemExit is a `BaseException`).
- **Reads environment variables**: `REASONS_PG_CONNINFO`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID`. No mutation.
- No filesystem or DB access here.

### Error Handling

- **No exceptions raised for validation failure** — the "error path" is a printed message + `SystemExit(1)`.
- **`AttributeError` risk**: if `args` lacks `db` and pg is not selected, the last line crashes. Nothing in the function protects against this; the top-level parser is assumed to have set `db`.
- **No pg connection-string parsing/validation** — format errors surface later from the pg driver.
- **Misleading message text**: the emitted error says `--domain-id is required with --pg`, but a user could have satisfied it via `--project-id`, `REASONS_DOMAIN_ID`, or `REASONS_PROJECT_ID`. The message is technically inaccurate for those cases.

### Usage Patterns

Called as `_backend_kwargs(args)` from most `cmd_*` functions, always immediately following construction of the rest of the API arguments and always `**`-expanded into the API call. The expectation is that the caller is inside a subcommand handler and that `main()` has already parsed global `--pg`, `--domain-id`, `--project-id`, and `--db` into the same `Namespace`.

Subcommands that do **not** call it use `_require_sqlite(args, name)` first (e.g., `check-integrity`, `derive`, `ask`, `build-wiki`, `review-*`) or call `db_path=args.db` directly (`mark-duplicate`, `mark-superseded`, `defeat-justification`, `migrate-defeaters`, `classify-defeaters`, `pin-*`, `contradictions`, `repair*`, `propose-update`). This is the observable pattern in `tests/test_pg_dispatch.py`.

Caller obligations:
- Must have verified the subcommand supports pg (or accept that a pg env var will silently change behavior).
- Must not attempt to inspect a return value on the error path — none exists.
- Must accept that `args.db` is only consulted when pg is absent.

### Dependencies

- `os` (env lookups).
- `sys` (stderr, exit).
- No dependency on `reasons.api`; this helper is pure dispatch logic.
- Implicitly depends on `main()`'s argparse setup: the top-level parser must define `--db`, `--pg`, `--domain-id`, and `--project-id` so the attributes exist, and the subcommand handlers must be reached through the same `Namespace`.

### Assumptions not enforced by the type system

- `args.db` is a valid SQLite path.
- `args.pg` (if set) is a parseable PostgreSQL DSN.
- Whether an API function actually implements a pg backend is checked elsewhere via `_require_sqlite`; this helper has no such guard, so a caller using `_backend_kwargs` against an unimplemented pg command will reach a KeyError/late failure rather than an early error.
- `did` is a string. Nothing enforces this — `Namespace` construction and env vars are the only sources, so this survives only by convention.
- The deprecated `--project-id`/`REASONS_PROJECT_ID` remain supported indefinitely with no runtime warning.

## Topics to Explore

- [file] `reasons/api.py` — Shows how the returned `db_path` or `pg_conninfo`+`domain_id` keys are dispatched to the SQLite vs. PostgreSQL backend implementation.
- [file] `tests/test_pg_dispatch.py` — Confirms the expected role of `_backend_kwargs`, the `_require_sqlite` guard, and the env-var precedence rules.
- [function] `reasons/cli.py:_require_sqlite` — Companion helper that explicitly rejects pg for commands that don't implement the pg backend; contrast with `_backend_kwargs` which silently selects pg.
- [general] `env-var-precedence-rules` — The specific ordering of `--domain-id > --project-id > REASONS_DOMAIN_ID > REASONS_PROJECT_ID` and the implication for tooling that populates both.
- [general] `deprecated-project-id-alias` — Whether and where `--project-id` / `REASONS_PROJECT_ID` are meant to be withdrawn; a search for `project_id` in `reasons/` reveals every use site.

## Beliefs

- `_backend-kwargs-requires-domain-id-when-pg` — When `--pg` or `REASONS_PG_CONNINFO` is set, `_backend_kwargs` requires a domain ID (via `--domain-id`, deprecated `--project-id`, or the corresponding env vars) and exits with code 1 if none is present.
- `_backend-kwargs-returns-one-of-two-shapes` — The function always returns either `{"db_path": ...}` or `{"pg_conninfo": ..., "domain_id": ...}` — never a mixed dict.
- `_backend-kwargs-pg-precedence-over-db` — When PostgreSQL is selected, `args.db` is silently ignored and not included in the returned kwargs.
- `_backend-kwargs-cli-args-override-env-vars` — Each backend config value prefers CLI arguments over environment variables (`--pg` > `REASONS_PG_CONNINFO`; `--domain-id`/`--project-id` > `REASONS_DOMAIN_ID` > `REASONS_PROJECT_ID`).
- `_backend-kwargs-exits-via-sys-exit` — On its validation-failure path, the function calls `sys.exit(1)` after writing to stderr, so callers cannot handle the failure with a normal `try/except Exception`.
