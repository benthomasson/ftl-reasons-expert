---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: cmd_classify_defeaters in reasons/cli.py

## Purpose

`cmd_classify_defeaters` is the CLI handler for `reasons classify-defeaters`. It invokes the backend API to classify existing defeaters by logical failure mode (`defeat_reason_type`), then formats the API result for terminal output.

It exists to provide a safe, batch-oriented CLI path for LLM-assisted defeater classification. By default it runs in dry-run mode, and it explicitly rejects the PostgreSQL backend.

## Contract

Preconditions:

- Called from the CLI dispatcher with an `argparse.Namespace`.
- The namespace must contain:
  - `args.apply`
  - `args.model`
  - `args.timeout`
  - `args.db`
- `args.pg` may be absent, but if present and truthy, the command refuses to run.
- `REASONS_PG_CONNINFO` must not be set to a truthy value.
- A SQLite database path must be supplied via `args.db`.
- The backend function `api.classify_defeat_reason_types` must exist and return a dict with `classified`, `skipped`, and `errors` keys.

Postconditions:

- The command prints a report to stdout.
- In dry-run mode, it prints `Dry run — no changes applied`.
- In apply mode, it prints `Classified: N` instead of `Would classify: N`.
- It prints skipped and error entries if present.
- It prints `No unclassified defeaters found` only when all three result lists are empty.
- The function returns `None`.

Important invariants:

- `dry_run = not args.apply`; absence of `--apply` means dry-run.
- The PG guard runs before the API call.
- The API is called exactly once.
- Only SQLite is supported, even though many other commands use `_backend_kwargs(args)`.
- The final “No unclassified defeaters found” message is not printed when there are skipped entries or errors, because the condition requires all three lists to be empty.

## Parameters

`args` is an `argparse.Namespace` created by the parser for the `classify-defeaters` subcommand.

Relevant fields:

- `args.pg` — optional PostgreSQL connection string. Missing is OK because it uses `getattr(args, "pg", None)`.
- `REASONS_PG_CONNINFO` — environment variable checked as an alternative PG trigger.
- `args.apply` — boolean from `--apply`. If false or absent, `dry_run` is true.
- `args.type` — optional defeater-type filter from `--type`. Accessed with `getattr(args, "type", None)`.
- `args.model` — required LLM model string from `--model/-m`.
- `args.timeout` — LLM timeout in seconds. Parser default is `300`.
- `args.db` — SQLite database path from the global `--db` option.

Assumptions not enforced by this function:

- `args.apply` is assumed to exist because it is accessed directly as `args.apply`, not via `getattr`.
- `args.model`, `args.timeout`, and `args.db` are assumed to exist and be valid.
- `args.type` is passed as-is. The subparser does not restrict choices, so any string could reach the API.
- The API result is assumed to contain dict-like entries with specific keys.

## Return Value

Returns `None`.

The CLI dispatcher ignores the return value. The only process-level termination performed by this function is via `sys.exit(1)` when the PG backend is requested or configured.

## Algorithm

1. Check for PostgreSQL backend usage:
   - If `getattr(args, "pg", None)` is truthy, or `os.environ.get("REASONS_PG_CONNINFO")` is truthy:
     - Print `Error: classify-defeaters is not supported with --pg` to stderr.
     - Exit with status code `1`.
2. Compute `dry_run = not args.apply`.
3. Call:

   `api.classify_defeat_reason_types(...)`

   passing:
   - `defeater_type_filter=getattr(args, "type", None)`
   - `model=args.model`
   - `timeout=args.timeout`
   - `dry_run=dry_run`
   - `db_path=args.db`

4. If `dry_run` is true:
   - Print `Dry run — no changes applied`
   - Print a blank line.
5. If `result["classified"]` is non-empty:
   - Print `Would classify: N` in dry-run mode, or `Classified: N` in apply mode.
   - Print each classified item as `  {id}: {defeat_reason_type}`.
6. If `result["skipped"]` is non-empty:
   - Print `Skipped: N`.
   - Print each skipped item as `  {id}: {reason}`.
7. If `result["errors"]` is non-empty:
   - Print `Errors: N`.
   - Print each error as `  {id}: {reason}`.
8. If all three result lists are empty:
   - Print `No unclassified defeaters found`.

Notable ordering detail: the dry-run banner is printed after the API call, not before it. If the API invokes an LLM, the user may not see the banner until classification finishes.

## Side Effects

- Reads the `REASONS_PG_CONNINFO` environment variable.
- Writes output to stdout.
- Writes an error to stderr and terminates the process with status `1` for unsupported PG usage.
- Calls `api.classify_defeat_reason_types`, which may:
  - Query the SQLite database.
  - Invoke an LLM.
  - Update defeater records when `dry_run` is false and `--apply` was passed.
- Does not write files, log to disk, or directly mutate database state itself.

## Error Handling

Handled errors:

- PostgreSQL requested via `--pg` or `REASONS_PG_CONNINFO`:
  - Prints a clear error to stderr.
  - Calls `sys.exit(1)`.

Unhandled errors:

- Exceptions raised by `api.classify_defeat_reason_types` are not caught. These propagate out of the command handler and would normally cause a traceback and non-zero CLI failure.
- Missing result keys such as `result["classified"]` raise `KeyError`.
- Malformed result entries, such as a classified item without `defeat_reason_type`, raise `KeyError`.
- Missing required namespace fields such as `args.apply` raise `AttributeError`.

Application-level errors returned inside `result["errors"]` are printed, but they do not cause a non-zero exit status. The command can print errors and still exit successfully.

## Usage Patterns

Typical dry-run invocation:

```bash
reasons --db reasons.db classify-defeaters -m claude
```

Typical apply invocation:

```bash
reasons --db reasons.db classify-defeaters -m claude --apply
```

Filter by defeater type:

```bash
reasons --db reasons.db classify-defeaters -m claude --type invalid-inference --apply
```

Caller obligations:

- Do not use this command with `--pg` or `REASONS_PG_CONNINFO`.
- Provide an LLM model.
- Use `--apply` only when database mutation is intended.
- Inspect printed `Errors:` entries manually; the command does not fail the process because of them.
- If custom error handling or exit codes are needed, call `api.classify_defeat_reason_types` directly instead of this CLI wrapper.

## Dependencies

- `os` — environment variable check for `REASONS_PG_CONNINFO`.
- `sys` — stderr output and process exit.
- `reasons.api` — specifically `api.classify_defeat_reason_types`.
- Parser wiring in `reasons/cli.py:main`, which defines the `classify-defeaters` subcommand and its arguments.

## Topics to Explore

- [function] `reasons/api.py:classify_defeat_reason_types` — Confirm exact dry-run semantics, whether it invokes the LLM in dry-run mode, and the precise result dict shape.
- [file] `tests/test_cli.py` — See how the CLI output, PG rejection, dry-run behavior, and result formatting are tested.
- [general] `pg-backend-support` — Understand which commands support PostgreSQL and why this command explicitly rejects it.
- [general] `dry-run-llm-behavior` — Determine whether dry-run prevents only database writes or also avoids model calls.
- [function] `reasons/cli.py:cmd_migrate_defeaters` — Compare its similar classified/skipped/errors reporting pattern with this command.

## Beliefs

- `classify-defeaters-rejects-pg-backend` — If `args.pg` or `REASONS_PG_CONNINFO` is truthy, `cmd_classify_defeaters` prints an error to stderr and exits with status 1 before calling the API.
- `classify-defeaters-defaults-to-dry-run` — `cmd_classify_defeaters` sets `dry_run = not args.apply`, so omitting `--apply` causes dry-run mode.
- `classify-defeaters-requires-result-buckets` — The function directly indexes `result["classified"]`, `result["skipped"]`, and `result["errors"]`, so the API must return all three keys.
- `classify-defeaters-does-not-fail-on-item-errors` — Entries in `result["errors"]` are printed but do not cause a non-zero exit status.
- `classify-defeaters-no-candidates-requires-empty-buckets` — The message `No unclassified defeaters found` is printed only when all three result lists are empty.
