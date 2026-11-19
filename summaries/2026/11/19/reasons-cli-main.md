---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: main in reasons/cli.py

# `main` — CLI Entry Point for `reasons`

## Purpose

`main()` is the top-level console-script entry point for the `reasons` command-line tool. It is responsible for:

1. Building the entire `argparse` grammar for the CLI (global flags + ~70 subcommands)
2. Parsing `sys.argv`
3. Rejecting invocations that omit a subcommand
4. Dispatching to the corresponding `cmd_*` handler function

Its job is purely **argument plumbing and dispatch**. Every command handler is a thin wrapper that calls into `reasons.api` or helper modules (`ask`, `derive`, `cluster`, `llm`, etc.) and formats the result for the terminal. No belief-network logic lives here.

## Contract

**Preconditions**
- The process's `sys.argv` is set up as `reasons [<global-options>] <command> [<command-options>]`.
- Global options must precede the subcommand (an argparse property; not enforced by this code — see "Assumptions" below).
- The installed distribution `ftl-reasons` must be resolvable on `importlib.metadata` (needed by `--version`).
- The environment may set `REASONS_PG_CONNINFO`, `REASONS_DOMAIN_ID`, or `REASONS_PROJECT_ID` (consumed later in `_backend_kwargs`).

**Postconditions**
- Either a `cmd_*` handler is executed and its side effects (stdout, DB writes, file writes, subprocess/LLM calls) are performed, or the process exits.
- Returns `None` on a normal, successful command completion. Callers (the console-script shim) ignore the return value; success is signaled by process exit code 0 (implicit).

**Invariants / exit codes**
- Successful command → exit 0 (default Python process exit).
- `--version` or `-h` → argparse prints and exits 0.
- Invalid arguments → argparse prints usage to stderr and exits 2.
- No subcommand → `parser.print_help()` (stdout) + `sys.exit(1)`.
- Handlers regularly call `sys.exit(1)` themselves on domain errors (e.g. `KeyError`, missing files, unsupported `--pg` combinations).
- Any *uncaught* exception raised inside a `cmd_*` handler propagates out of `main()` as a traceback, exit code 1 (via Python's default behavior). `main()` does **not** wrap dispatch in try/except.

## Parameters

`main()` takes **no parameters**. All input comes from `sys.argv`.

Notable global flags declared on the root parser:

| Flag | Meaning |
|---|---|
| `--version` | Prints `reasons <pkg_version("ftl-reasons")>` and exits. |
| `--db` | SQLite DB path; default `api.DEFAULT_DB`. |
| `--pg CONNINFO` | PostgreSQL connection string; alternative backend to SQLite. |
| `--domain-id` | Required when `--pg` is set; otherwise unused. |
| `--project-id` | Deprecated alias for `--domain-id`. |

`--db` and `--pg` are **not** declared mutually exclusive; the resolution is deferred to `_backend_kwargs`.

Every subcommand receives the same `argparse.Namespace` object populated by the top-level parser *plus* its own subparser's arguments.

## Return Value

None. `main()` has no return value — it either dispatches or exits the process.

Callers of the CLI rely on process exit codes:
- `0` — success (default)
- `1` — explicit failure (`sys.exit(1)` from `main()` or a handler, or uncaught exception)
- `2` — argparse syntax error

## Algorithm

**Step 1 — Build the root parser**

```python
parser = argparse.ArgumentParser(prog="reasons", description=...)
parser.add_argument("--version", action="version", version=f"%(prog)s {_pkg_version('ftl-reasons')}")
parser.add_argument("--db", ...)  # etc.
sub = parser.add_subparsers(dest="command")
```

Key point: `_pkg_version('ftl-reasons')` is **evaluated eagerly** at parser-construction time (the f-string is evaluated before `add_argument`). So `main()` will raise `PackageNotFoundError` (from `importlib.metadata`) *before* argparse can show help if the package metadata is missing. That is why `--version` shows the real installed version.

`subparsers(dest="command")` stores the chosen subcommand's token in `args.command` — or `None` if none is given. `required=True` is not set here; the check is manual.

**Step 2 — Register every subcommand**

The code follows a mechanical pattern:

```python
p = sub.add_parser("add", help="Add a node")
p.add_argument("node_id", ...)
p.add_argument("text", ...)
...
p = sub.add_parser("retract", help="...")
p.add_argument("node_id", ...)
```

The local variable `p` is repeatedly rebound to a new `ArgumentParser` (each subparser), so subsequent `p.add_argument(...)` calls attach to the *latest* subcommand.

Two subcommands redeclare global-looking option names:
- `search-sources` declares `--db` (required) as the **FTS5 chunk-DB path** — not the reasons DB. This shadows the top-level `--db` for that subparser's namespace.
- `supersede --id` vs. `propose-supersede --new-id` vs. `supersede --text` follow independent naming conventions.

Subparsers without any options are added directly:

```python
sub.add_parser("propagate", ...)
sub.add_parser("repos", ...)
sub.add_parser("namespaces", ...)
```

**Step 3 — Parse**

```python
args = parser.parse_args()
```

Argparse parses global opts, then the subcommand token, then delegates to the appropriate subparser to fill in the remaining namespace. On any syntax error, argparse itself prints usage + error and calls `sys.exit(2)`.

**Step 4 — Guard against missing subcommand**

```python
if not args.command:
    parser.print_help()
    sys.exit(1)
```

Help goes to **stdout**, but the exit code is 1 — semantically signaling "you forgot a subcommand."

**Step 5 — Dispatch via dictionary**

```python
commands = { "init": cmd_init, "add": cmd_add, ... }
commands[args.command](args)
```

The dictionary is constructed fresh on every invocation. Simple, explicit, and avoids `set_defaults(func=...)` — but it means the parser grammar and the dispatch table are two **parallel lists** that must be kept in sync by hand (see Assumptions).

Aliases are expressed as duplicate dict entries:
- `"research": cmd_repair` (deprecated alias for `repair`)
- `"pull": cmd_pull` — where `cmd_pull` itself just calls `cmd_import_hf(args)`

## Side Effects

Within `main()` itself:

- **`_pkg_version('ftl-reasons')`** — reads installed distribution metadata at parser-construction time.
- **`parser.print_help()`** to stdout when no subcommand.
- **`sys.exit(1)`** when no subcommand.
- **`parser.parse_args()`** — writes help/error output to stdout/stderr and calls `sys.exit(0|2)` in response to `--help`, `--version`, or bad input.
- **`commands[args.command](args)`** — invokes the handler, which performs all real work (DB I/O, file writes, LLM subprocess invocations, HTTP calls, etc.).

`main()` does **not** touch the DB, filesystem, or environment directly, nor does it validate backend configuration — that happens lazily inside handlers via `_backend_kwargs(args)` / `_require_sqlite(args, name)`.

## Error Handling

`main()` itself only raises via:

- `PackageNotFoundError` at parser construction if `ftl-reasons` metadata is missing (bad install).
- `SystemExit` from `sys.exit(1)` (missing subcommand) and from argparse (`-h`, `--version`, bad args).
- `KeyError` if dispatch is reached with a command name that is not a dict key — **theoretically possible only if a subparser was added but not added to the dict**, or the dict is updated and the subparser is not. This is not caught; it would propagate as a traceback.
- Any exception raised inside a handler — inherits whatever `cmd_*` does. Handlers generally catch `KeyError`, `ValueError`, `FileNotFoundError`, `IndexError`, `RuntimeError`, and call `sys.exit(1)` with a formatted message. Uncaught exceptions produce a Python traceback and exit code 1.

Errors are **not** swallowed or converted to return codes — `main()` has no return value; exit status is the only signal.

## Usage Patterns

- **Console script**: `pyproject.toml` typically declares `reasons = "reasons.cli:main"`, so `pip install` exposes `reasons` on `$PATH`. Users run:
  ```
  reasons --db reasons.db status --all
  reasons add my-node "some belief" --sl a,b
  reasons --pg "postgresql://..." --domain-id mydomain status
  ```
- **Tests** (`tests/test_cli.py`) typically monkeypatch `sys.argv` and call `main()` directly, or invoke handlers to bypass argparse.
- **Global flags must be given before the subcommand** because of how argparse composes parsers: `reasons --db foo status` works; `reasons status --db foo` does not (unless the subparser re-declares `--db`, which most do not).

Caller obligations:
- You must be on a system where `reasons` is imported as a package (or use `python -m reasons.cli`).
- Provide `--domain-id` alongside `--pg` (or set `REASONS_DOMAIN_ID`) for any handler calling `_backend_kwargs`.
- Prefer to check `sys.exit` semantics over return values: the CLI has no structured result object.

## Dependencies

- **stdlib**: `argparse`, `sys`, `os` (indirectly through helpers), `importlib.metadata.version as _pkg_version`
- **`reasons.api`** — provides all domain operations (`api.init_db`, `api.add_node`, `api.retract_node`, `api.DEFAULT_DB`, etc.) that `cmd_*` wrappers call.
- **`reasons.llm`**, **`reasons.ask`**, **`reasons.derive`**, **`reasons.cluster`**, **`reasons.mcp_client`**, **`reasons.propose_update`** — imported lazily inside handlers, not at module import time.
- Every `cmd_*` function defined in the same file — `main` dispatches to them.

## Assumptions Not Enforced by the Type System

1. **Grammar / dispatch table sync**: Two parallel lists — the `sub.add_parser(...)` calls on one side and the `commands = {...}` dict on the other — must expose the same command set. Forgetting to update one side yields either a `KeyError` at dispatch (parser knows a command, dict doesn't) or dead code (dict knows a command, parser can't reach it). No test or type annotation enforces this.

2. **Global-then-subcommand ordering**: Argparse places global options on the root parser, so they cannot appear after the subcommand. Users frequently expect otherwise; nothing in `main` warns about it.

3. **`--db` / `--pg` mutual exclusivity**: Not declared via `mutually_exclusive_group`. Instead, `_backend_kwargs` prefers `--pg` and silently ignores `--db` when both are present.

4. **`search-sources --db` shadowing**: `search-sources` re-declares `--db` (required, FTS chunk DB). Passing `--db reasons.db search-sources --db rag_fts.db` will end up with `args.db == "rag_fts.db"` in the handler namespace — the top-level value is overwritten. This is arguably the intended design (the two DBs are conceptually distinct) but is not documented.

5. **`ftl-reasons` package name**: `--version` hard-codes the distribution name `ftl-reasons`. Renaming the package would break `--version` with a `PackageNotFoundError` even if the code runs.

6. **Handler return values are ignored**: `commands[args.command](args)` discards whatever the handler returns. Handlers must call `sys.exit(1)` on failure — they cannot return an error code.

7. **`cmd_*` handler signature**: Handlers are assumed to accept exactly one positional argument (`args: argparse.Namespace`). This is implied by the dict but not typed. Handlers for aliased commands (`research`, `pull`) must tolerate the exact `Namespace` shape built by the subparser they were registered under (e.g. `research` shares `cmd_repair`'s shape; `pull` shares `cmd_import_hf`'s).

## Topics to Explore

- [file] `reasons/api.py` — Every `cmd_*` handler ultimately delegates to functions here; the CLI's real behavior lives in this layer.
- [function] `reasons/cli.py:_backend_kwargs` — Central place where `--db` vs `--pg` is resolved and where the `--domain-id` precondition is enforced (exits 1 if missing).
- [function] `reasons/cli.py:cmd_search_sources` — Concrete example of a handler whose `--db` shadows the global, illustrating the loose coupling between the root parser and subparsers.
- [file] `reasons/ask.py` — Provides `ask()` and `search_source_chunks()` used by the `ask` and `search-sources` commands; shows how LLM synthesis is orchestrated.
- [general] `cli-grammar-dispatch-parity` — How to verify that every `sub.add_parser(...)` name has a matching entry in the `commands` dict; current code relies on discipline rather than automated checks.

## Beliefs

- `main-requires-global-flags-before-subcommand` — Global options (`--db`, `--pg`, `--version`, etc.) must precede the subcommand token because they are declared on the root parser, not passed through to subparsers.
- `backend-kwargs-exits-on-pg-without-domain-id` — `_backend_kwargs` calls `sys.exit(1)` if `--pg` (or `REASONS_PG_CONNINFO`) is set without any `--domain-id` / `--project-id` / env fallback.
- `search-sources-db-shadows-global-db` — The `search-sources` subparser re-declares `--db` as a required FTS5 chunk-DB path, overwriting the globally-scoped `--db` in the parsed namespace.
- `main-exits-1-on-missing-subcommand` — When `args.command` is `None`, `main` prints the help to stdout and calls `sys.exit(1)` rather than relying on argparse's `required=True`.
- `command-dispatch-uses-parallel-dict` — Dispatch is implemented via a hand-maintained `commands` dictionary, not `subparsers.set_defaults(func=...)`; parser registration and dispatch entries are two parallel lists with no automatic synchronization check.
