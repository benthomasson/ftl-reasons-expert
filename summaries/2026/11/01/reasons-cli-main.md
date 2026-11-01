---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Function: main in reasons/cli.py

# `main()` — CLI Entry Point for the Reason Maintenance System

## 1. Purpose

`main()` is the **sole entry point** for the `reasons` command-line tool. It builds a complete `argparse` argument parser with ~80 subcommands, parses the user's command-line invocation, and dispatches to the appropriate `cmd_*` handler function defined in the same file. It exists to provide a single, flat CLI surface over the `reasons.api` module, which implements the Reason Maintenance System (RMS) — a truth-maintenance system that tracks beliefs (nodes), their justifications, and dependency-directed backtracking.

The function has no business logic of its own; it is purely a **routing and argument-validation layer**.

## 2. Contract

| Aspect | Detail |
|--------|--------|
| **Precondition** | `sys.argv` must contain a valid subcommand name (or the function will exit with code 1 after printing help). The `reasons.api` module must be importable. |
| **Postcondition** | Exactly one `cmd_*` function is called with the parsed `args` namespace. The process exits with the exit code set by that handler (or 0 if the handler returns normally without calling `sys.exit`). |
| **Invariant** | The `commands` dict keys are the exact strings passed to `sub.add_parser(...)`. If a subcommand is registered in argparse but missing from the dict, `args.command` will raise a `KeyError` (unhandled). |

## 3. Parameters

The function takes **no parameters**. All input comes from `sys.argv` via `argparse`.

**Global (top-level) arguments:**

| Flag | Default | Purpose |
|------|---------|---------|
| `--version` | — | Prints `reasons <version>` (from `importlib.metadata`) and exits |
| `--db` | `api.DEFAULT_DB` (typically `"reasons.db"`) | Path to the SQLite database file |
| `--pg` | `None` | PostgreSQL connection string; when set, overrides `--db` for PG-backed commands |
| `--domain-id` | `None` | PostgreSQL domain/schema identifier (required alongside `--pg`) |
| `--project-id` | `None` | Deprecated alias for `--domain-id` |

**Subcommand arguments:** Each subcommand defines its own positional and optional arguments. These are registered via `sub.add_parser(...)` and their values land on the `args` namespace alongside the global flags. The parsed namespace is passed opaquely to the handler; each `cmd_*` function uses `getattr(args, ...)` to extract only the fields it needs.

**Edge cases:**
- Running `reasons` with no subcommand → prints help, exits 1.
- Running `reasons --pg CONNINFO retract node-1` without `--domain-id` → the `_backend_kwargs` helper (called inside the handler) will exit 1 with an error message.
- `--project-id` is silently accepted but is a deprecated no-op alias; `_backend_kwargs` checks `domain_id` first, then falls back to `project_id`.

## 4. Return Value

The function **never returns a value**. It either:
- Calls `sys.exit(1)` (when no command is given), or
- Calls the dispatched `cmd_*` function, which itself may call `sys.exit(n)` on error, or
- Falls through to implicit `None` return (process exits 0).

The caller (typically a `console_scripts` entry point in `pyproject.toml` or `setup.py`) invokes `main()` and relies on the process exit code for success/failure signaling.

## 5. Algorithm

```
1. Create ArgumentParser (prog="reasons")
2. Register global flags: --version, --db, --pg, --domain-id, --project-id
3. Create subparsers (dest="command")
4. For each of ~80 subcommands:
     a. sub.add_parser("<name>", help=...)
     b. Register its positional and optional arguments
5. args = parser.parse_args()
6. IF args.command is None:
     a. parser.print_help()
     b. sys.exit(1)
7. Build `commands` dict: {"init": cmd_init, "add": cmd_add, ...}
8. commands[args.command](args)   ← single dispatch call
```

The dispatch is a **flat dictionary lookup** — there is no class hierarchy, no plugin system, no dynamic loading. Every handler must be defined in this file and listed in the dict.

A notable detail: `"research": cmd_repair` — the `research` subcommand is a **deprecated alias** that maps to the same handler as `repair`. This is the only alias in the dispatch table.

## 6. Side Effects

`main()` itself performs **no I/O, no mutations, no external calls**. Its only side effects are:

- Reading `sys.argv` (via `argparse`)
- Printing help text to stdout (if no command is given)
- Calling `sys.exit(1)` (if no command is given)
- Invoking exactly one `cmd_*` function, which may perform arbitrary I/O (DB reads/writes, LLM calls, file writes, network requests to HuggingFace, etc.)

The `argparse` setup is purely in-memory object construction.

## 7. Error Handling

| Scenario | Behavior |
|----------|----------|
| No subcommand provided | `parser.print_help()` to stdout, `sys.exit(1)` |
| Unknown subcommand | `argparse` prints an error to stderr, exits 2 (argparse's default) |
| Invalid argument values (e.g., `--limit abc`) | `argparse` prints an error to stderr, exits 2 |
| `args.command` not in `commands` dict | **Unhandled `KeyError`** — this is a latent bug path. In practice it can't happen because both the subparser registration and the dict are maintained by the same developer, but there's no static check. |
| `--version` flag | `argparse` prints version, exits 0 |

The function does **not** wrap the dispatch in a try/except. Any uncaught exception in a `cmd_*` handler will propagate as a Python traceback to stderr with exit code 1.

## 8. Usage Patterns

This is invoked as a **console script**:

```bash
reasons add my-node "The sky is blue" --sl premise-1,premise-2
reasons retract my-node --reason "Actually it's green"
reasons search "kubernetes networking" --limit 10
reasons derive --model claude --budget 200 --auto
reasons --pg "host=pg.example.com dbname=reasons" --domain-id prod-42 status --all
```

It's also called programmatically in tests (e.g., `tests/test_cli.py`) by monkeypatching `sys.argv` and calling `main()`, then catching `SystemExit`.

**Caller obligations:**
- The `reasons` package must be installed (for `importlib.metadata.version('ftl-reasons')` to work).
- The working directory or `--db` path must point to a valid SQLite file (or `--init` must be run first).
- For PostgreSQL mode, `--pg` and `--domain-id` must both be provided.

## 9. Dependencies

| Module | Usage |
|--------|-------|
| `argparse` | CLI parsing (stdlib) |
| `sys` | Exit codes, stderr |
| `importlib.metadata` | Package version lookup for `--version` |
| `reasons.api` (`.api`) | All business logic; `api.DEFAULT_DB` for the default path |
| `reasons.ask` | Imported lazily inside `cmd_ask` and `cmd_search_sources` |
| `reasons.llm` | Imported lazily inside LLM-dependent handlers |
| `reasons.derive` | Imported lazily inside `cmd_derive` / `cmd_accept` |
| `reasons.cluster` | Imported lazily for embedding-based clustering |
| `reasons.mcp_client` | Imported lazily for MCP server bridges in `cmd_ask` |

The `main()` function itself only depends on `argparse`, `sys`, `importlib.metadata`, and `api.DEFAULT_DB`. All other imports are deferred to the individual `cmd_*` handlers.

**Assumptions not enforced by the type system:**
- `args.command` is guaranteed to be a key in `commands` — this is only true by convention (both lists are hand-maintained). A new subcommand added to argparse but forgotten in the dict will crash at runtime.
- `api.DEFAULT_DB` is a string path — if it were `None` or a `Path`, `argparse` would accept it but downstream code might break.
- The `_pkg_version('ftl-reasons')` call assumes the package is installed in a way that `importlib.metadata` can find it (e.g., pip-installed, not just on `sys.path`).

---

## Topics to Explore

- [file] `reasons/api.py` — The entire backend that every `cmd_*` function delegates to; understanding its function signatures and return shapes is essential to understanding what the CLI is wrapping
- [file] `tests/test_cli.py` — Shows how `main()` is exercised in tests, including how `sys.argv` is monkeypatched and how `SystemExit` is caught
- [function] `reasons/cli.py:_backend_kwargs` — The small helper that resolves `--pg`/`--domain-id`/`--db` into the correct kwargs dict passed to every API call; critical for understanding the dual-backend (SQLite vs PostgreSQL) design
- [general] `truth-maintenance-system` — The theoretical foundation (TMS, dependency-directed backtracking, outlists, nogoods) that the `add`/`retract`/`assert`/`defeat-justification` commands implement
- [file] `reasons/derive.py` — The prompt-building, proposal-parsing, and apply pipeline that `cmd_derive` orchestrates; the most complex handler in the CLI

---

## Beliefs

- `main-dispatches-via-flat-dict` — `main()` routes subcommands through a single hand-maintained dictionary; adding a subparser without a matching dict entry causes an unhandled `KeyError` at runtime (no static check or test guards this)
- `research-alias-maps-to-cmd-repair` — The `research` subcommand is a deprecated alias that dispatches to `cmd_repair`, not a separate implementation; both share identical argument definitions
- `main-performs-no-business-logic` — `main()` contains zero I/O, zero DB access, and zero mutation; all side effects are deferred to the `cmd_*` handlers it dispatches to
- `dual-backend-selected-per-invocation` — The `--pg` flag (or `REASONS_PG_CONNINFO` env var) switches the backend from SQLite to PostgreSQL on a per-invocation basis via `_backend_kwargs`, which is called inside each handler rather than once in `main()`
- `exit-code-1-signals-usage-error` — When no subcommand is provided, `main()` exits with code 1 (not 0), signaling a usage error to the shell; `argparse` itself exits with code 2 for malformed arguments
