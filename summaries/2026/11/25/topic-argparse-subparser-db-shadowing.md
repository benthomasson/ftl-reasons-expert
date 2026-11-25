---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The root cause of the "which `--db` is which?" confusion; worth understanding before adding new subcommands that also want their own DB argument.

# Root Cause: "Which `--db` is Which?"

## Context

`reasons/cli.py` is a 4,015-line module (per the observation block) whose handlers are thin wrappers over `reasons.api`. `reasons_lib/cli.py` is only 3 lines and does `from reasons.cli import *` — so the two import paths are the *same* CLI, not two.

## The Root Cause

The CLI has **two different ways of resolving the database, and neither is visible in the observed window**. That ambiguity is the bug you're worrying about.

### 1. Two DB-resolution paths coexist in the handlers

Look at what the returned slice of `reasons/cli.py` shows:

- `cmd_init`, `cmd_add`, `cmd_add_justification`, `cmd_remove_justification`, `cmd_retract` all call **`**_backend_kwargs(args)`** and never touch `args.db` directly.
- `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters` all bypass that helper and pass **`db_path=args.db`** directly (e.g. `reasons/cli.py:cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`).

So `--db` is not "owned" by one layer. Some subcommands route through a helper that presumably translates `args.db` / `args.pg` / env vars into backend kwargs; others short-circuit straight to `args.db`. When you ask "which `--db` is which," the honest answer today is: *it depends on which handler you're reading*.

### 2. The SQLite-only guard is duplicated, not centralized

Every handler that short-circuits on `args.db` re-implements the same guard:

```python
if getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO"):
    print("Error: ... is not supported with --pg ...", file=sys.stderr)
    sys.exit(1)
```

This is copy-pasted five times (once each in `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`). That means the "SQLite only" contract is enforced per-command instead of centrally — which is exactly the kind of drift you get when it's ambiguous whether `--db` and `--pg` are mutually exclusive alternatives, or whether `--db` is a sub-flag of the SQLite backend selected via `--pg`.

### 3. The actual `--db` argparse definition is *not in the observations*

Both `reasons/cli.py` observation sections that should locate it failed to return anything:

- `cli_db_argument_occurrences` reports `total_count: 0`
- `subparser_definitions` reports `total_count: 0` for `add_subparsers|add_parser|set_defaults|ArgumentParser`

That is a false negative — `argparse` is imported at the top of `reasons/cli.py` and `args.db`/`getattr(args, "pg", ...)` are dereferenced, so the parser construction exists. It lives **past line 300** of a 4,015-line file. Because the observation tool only returned the first ~300 lines, we cannot see:

- whether `--db` is added on the **root parser** (inherited by all subcommands) or **per-subparser** (where a new subcommand's `--db` would silently shadow the parent),
- whether `--pg` is a sibling of `--db` on the same parser, a global flag, or an env-var-only switch,
- what `_backend_kwargs(args)` actually returns — presumably `{"db_path": ..., "pg": ...}` — and whether it reads env vars in addition to `args.db` / `args.pg`.

### 4. Two import surfaces amplify the risk

Because `reasons_lib/cli.py` is a wildcard re-export, any future "own DB argument" for a subcommand has to work through *both* `reasons.cli` and `reasons_lib.cli`. It almost certainly does (they're the same objects), but the surface area is confusing and a naive `git grep "--db"` will miss definitions that live in the tail of a 4k-line file.

### Why this matters *before* adding a new subcommand

If you add a subcommand that wants its own DB argument, you have to first decide — consistently with the existing mess — whether it:

1. **Inherits** the global `--db` and resolves through `_backend_kwargs(args)` (the `cmd_add` pattern), or
2. **Declares its own** `--db` on the subparser and hand-rolls the `args.pg`/`REASONS_PG_CONNINFO` guard (the `cmd_mark_*` pattern).

Pick the wrong one and `--db` will mean two different things depending on position on the command line, or the PG guard will silently not run, or `_backend_kwargs` will ignore your per-subcommand value. That is the confusion the question is pointing at.

## Topics to Explore

- [file] `reasons/cli.py` — Read lines 300–end, especially the `_backend_kwargs` definition and the `add_subparsers`/`add_parser` block, to see whether `--db` and `--pg` live on the root parser or shadow onto subparsers.
- [function] `reasons/cli.py:_backend_kwargs` — Determines how `args.db`, `args.pg`, and `REASONS_PG_CONNINFO` are combined into backend kwargs; new subcommands should match this contract or explicitly opt out.
- [file] `reasons/api.py` — The `api.*` functions accept `db_path=` and (somewhere) a PG alternative; verify whether every API entrypoint accepts both or only one.
- [file] `reasons/storage.py:Storage.__init__` — `Storage(db_path, project_name="")` takes `db_path` as a positional first arg, so confirm `_backend_kwargs` emits a `db_path` key compatible with this signature.
- [general] `sqlite-vs-pg-parity` — Audit which commands support PG (via `_backend_kwargs`) versus which are hard-blocked by the duplicated `args.pg` guard; this is the invariant a new subcommand must preserve.

## Beliefs

- `reasons-lib-cli-is-alias-of-reasons-cli` — `reasons_lib/cli.py` is a one-line `from reasons.cli import *`, so the two modules resolve to the same parser object and no independent `--db` can live there.
- `cli-handlers-use-two-competing-db-resolution-patterns` — `cmd_init`/`cmd_add`/`cmd_add_justification`/`cmd_remove_justification`/`cmd_retract` use `**_backend_kwargs(args)`; `cmd_mark_duplicate`/`cmd_mark_superseded`/`cmd_defeat_justification`/`cmd_defeat_with_scope`/`cmd_migrate_defeaters` use `db_path=args.db` directly.
- `pg-support-guard-is-duplicated-five-times` — The check `if getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO")` is repeated verbatim in five handlers rather than centrally, so each of those commands independently enforces the SQLite-only contract.
- `subparser-definitions-live-outside-observed-window` — `argparse` is imported and `args.db` is dereferenced in `reasons/cli.py`, yet the observation search returned 0 matches for both DB patterns and `add_subparsers`/`add_parser`, confirming the parser setup is in the unobserved tail of the 4,015-line file.
- `api-accepts-db-path-keyword` — Every `api.*` call in the observed slice passes the DB as `db_path=...` (or via `**_backend_kwargs`), so any new subcommand must emit a `db_path` key to match `Storage.__init__(self, db_path, ...)` in `reasons/storage.py`.
