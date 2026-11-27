---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: cmd_migrate_defeaters in reasons/cli.py

## `cmd_migrate_defeaters` — Explanation

### 1. Purpose

This is the CLI handler for the `reasons migrate-defeaters` subcommand. It is a **migration tool** that converts legacy string-based `retract_reason` metadata (stored on nodes) into **graph-native defeater beliefs** — i.e. it lifts reasons that used to be free-text metadata into actual nodes/invalid-inference defeaters that participate in the truth-maintenance graph.

It's a thin wrapper: it does the argument normalization, a backend guard, and formatting of the result dict returned by `api.migrate_retract_to_defeaters`. All actual migration logic lives in the API layer.

By default it is a **dry run**; nothing is mutated unless `--apply` is passed.

---

### 2. Contract

- **Precondition**: The command runs against the SQLite backend only. Postgres (`--pg` or `REASONS_PG_CONNINFO`) is not supported because there is no `PgApi` implementation of the migration.
- **Postcondition (dry run)**: A formatted report of what *would* migrate / be skipped / error is printed; the database is not modified.
- **Postcondition (`--apply`)**: The migration is applied; the same report is printed with "Migrated" rather than "Would migrate".
- **Invariants**:
  - Empty `node_ids` means "all candidates" (empty list normalizes to `None`).
  - Item-level failures in `result["errors"]` are reported but do **not** stop execution and do **not** set a non-zero exit code.
  - `defeater_id` is optional per migrated item; a fallback synthetic name is invented purely for display.

---

### 3. Parameters

`args` is an `argparse.Namespace` populated by the `migrate-defeaters` subparser:

| Attribute | Meaning | Type / Values |
|---|---|---|
| `args.node_ids` | Positional `nargs="*"` list of specific node IDs to migrate. Empty → `None` → migrate all candidates. | `list[str]` |
| `args.apply` | If false (default), dry run; if true, apply. | `bool` |
| `args.db` | Path to the SQLite database. | `str` |
| `args.pg` | Optional Postgres conninfo. If present or env is set, error out. | `str \| None` |

Edge cases:
- Passing zero positional args (`reasons migrate-defeaters`) → `node_ids = []` → `or None` → migrate everything.
- Passing `--apply` flips `dry_run = False`.
- `--pg` is checked before any API call, so no accidental Postgres attempt.

---

### 4. Return Value

Returns `None`. It is a command handler. On the pg guard it calls `sys.exit(1)`. Otherwise it just prints and falls through.

Caller (the `main()` dispatcher) does nothing with the return value. There is **no** non-zero exit on per-item migration errors — those are just printed.

---

### 5. Algorithm (step by step)

1. **Backend guard**: If `args.pg` is truthy or `REASONS_PG_CONNINFO` is set in the environment → print a clear error to `stderr` and `sys.exit(1)`.
2. **Normalize node IDs**: `node_ids = getattr(args, "node_ids", None) or None`. This maps `[]` → `None` (all candidates), and `["a", "b"]` → `["a", "b"]` (filtered).
3. **Determine dry run**: `dry_run = not args.apply` (default = dry run).
4. **Call the API**:
   ```python
   result = api.migrate_retract_to_defeaters(
       node_ids=node_ids, dry_run=dry_run, db_path=args.db)
   ```
   This is the only place the database may be mutated.
5. **Header**: If dry run, print `"Dry run — no changes applied"` and a blank line.
6. **Migrated section**: If `result["migrated"]` is non-empty:
   - Print `"Would migrate: N"` or `"Migrated: N"`.
   - For each item, resolve the defeater ID via `m.get("defeater_id", f"migrated-retraction-{m['id']}-j{m['justification_index']}")` — a defensive fallback so output is stable even if the api omitted the ID.
   - Print `"{id} j{justification_index} -> {defeater}"` and `"Reason: {retract_reason}"`.
7. **Skipped section**: If `result["skipped"]`, print count and `"{id}: {reason}"` for each.
8. **Errors section**: If `result["errors"]`, print count and `"{id}: {reason}"` for each.
9. **Empty case**: If none of the three lists has content, print `"No candidates found"`.

---

### 6. Side Effects

- Writes formatted lines to **stdout**.
- Writes error to **stderr** (pg guard) and calls `sys.exit(1)`.
- Calls `api.migrate_retract_to_defeaters`, which is the **only** side-effect path — and it only mutates the DB when `dry_run is False`.
- No file writes, no network, no global state.

---

### 7. Error Handling

- **Postgres rejection**: explicit `print(...)` + `sys.exit(1)`.
- **Per-item failures**: returned in `result["errors"]`; the function prints them but **does not** exit non-zero. Downstream scripts cannot rely on exit status to detect partial failures.
- **Uncaught exceptions**: `KeyError` for missing `result` keys, or API-raised exceptions (`ValueError`, `KeyError`, etc.) propagate out of the handler and will crash the CLI without a friendly message. There is no `try/except` around the API call here, unlike sibling commands such as `cmd_defeat_justification`.

---

### 8. Usage Patterns

```bash
# Preview what would be migrated (dry run by default)
reasons migrate-defeaters

# Migrate specific nodes
reasons migrate-defeaters my-node-1 my-node-2

# Actually apply
reasons migrate-defeaters --apply

# Apply for a subset
reasons migrate-defeaters my-node-1 --apply
```

Caller obligations:
- **Check command exit code before assuming success** is insufficient: per-item errors do not signal failure. Operators must read the printed `Errors:` section.
- Run the dry run first; the tool is designed to be inspected before applying.

Related sibling commands: `cmd_classify_defeaters` (classifies already-existing defeaters), `cmd_defeat_justification`, `cmd_defeat_with_scope`.

---

### 9. Dependencies

- `os`, `sys` (stdlib) — env var check, stderr, exit.
- `reasons.api.migrate_retract_to_defeaters` — the actual migration engine: returns the `{"migrated": [...], "skipped": [...], "errors": [...]}` dict.
- Wired into the command table in `main()` under key `"migrate-defeaters"`.

**Assumptions not enforced by the type system:**

1. `result` is a `dict` with exactly the keys `"migrated"`, `"skipped"`, `"errors"`.
2. Every `migrated` item contains `"id"`, `"justification_index"`, and `"retract_reason"` strings/ints; `defeater_id` is optional.
3. Every `skipped`/`errors` item contains `"id"` and `"reason"`.
4. `node_ids=None` unambiguously means "all candidates" in the API layer (not "none").
5. The defeater ID naming convention `migrated-retraction-{id}-j{index}` matches what the API would produce when it doesn't return one — otherwise the printed "->" target in a dry run might not match what `--apply` actually creates.

---

## Topics to Explore

- [function] `reasons/api.py:migrate_retract_to_defeaters` — The engine; defines what "candidate", "skipped", and "error" actually mean, and whether it truly matches the fallback ID naming.
- [function] `reasons/cli.py:cmd_classify_defeaters` — Sibling command that operates on defeaters already produced by this migration; shows the follow-up workflow.
- [file] `tests/test_cli.py` — Where the CLI contract (dry-run default, exit codes, output shape) is asserted; check whether per-item errors are tested for exit status.
- [general] `defeater-migration-strategy` — Design rationale for converting `retract_reason` metadata to graph-native defeaters (why the graph replaces strings).
- [function] `reasons/cli.py:_require_sqlite` — Reusable helper used by *other* commands for the same pg guard; useful to see whether `cmd_migrate_defeaters` diverges from the shared pattern.

## Beliefs

- `migrate-defeaters-is-dry-run-by-default` — `dry_run` defaults to `True` and is disabled only when `--apply` is passed.
- `migrate-defeaters-rejects-postgres` — The command calls `sys.exit(1)` if `--pg` or `REASONS_PG_CONNINFO` is set.
- `migrate-defeaters-empty-node-ids-means-all` — An empty positional list normalizes to `None` (`[] or None`), meaning "migrate all candidates".
- `migrate-defeaters-does-not-set-nonzero-exit-on-per-item-errors` — Item failures are printed from `result["errors"]` but do not change the process exit code.
- `migrate-defeaters-delegates-to-api` — All mutation and candidate selection is performed by `api.migrate_retract_to_defeaters`, not by this handler.
- `migrate-defeaters-has-fallback-defeater-id` — When `defeater_id` is absent, display falls back to `migrated-retraction-{id}-j{index}`.
