# File: reasons_lib/cli.py

**Date:** 2026-05-11
**Time:** 12:45

# `reasons_lib/cli.py` — CLI for the Reason Maintenance System

## 1. Purpose

This file is the **command-line interface** for `ftl-reasons`, a Truth Maintenance System (TMS). It is a pure presentation layer: it parses arguments via `argparse`, delegates every operation to `reasons_lib.api`, and formats the returned dicts for terminal output. It owns the `reasons` command entrypoint (registered via `pyproject.toml`) and defines ~40 subcommands.

The file is explicitly described in its docstring as "thin wrappers around `reasons_lib.api`" — it contains no business logic, no database access, and no network manipulation. All domain work lives in `api.py` and the modules it calls.

## 2. Key Components

### Entry point

- **`main()`** — Builds the top-level `argparse.ArgumentParser`, registers all subcommands, and dispatches via a `commands` dict mapping command name strings to `cmd_*` functions. Global flags include `--db`, `--pg`, `--project-id`, and `--version`.

### Backend routing

- **`_backend_kwargs(args)`** — Central function that inspects `--pg` / `REASONS_PG_CONNINFO` and `--project-id` / `REASONS_PROJECT_ID` to decide between SQLite (`{"db_path": ...}`) and PostgreSQL (`{"pg_conninfo": ..., "project_id": ...}`). Every `cmd_*` function spreads this into its `api.*` call.

- **`_require_sqlite(args, command_name)`** — Guard for commands that don't support PostgreSQL (e.g., `derive`, `ask`, `review-beliefs`, `deduplicate`, `contradictions`). Exits with an error if `--pg` is set.

### Command handlers (selected)

| Function | Subcommand | What it does |
|---|---|---|
| `cmd_add` | `add` | Creates a node (premise or derived) with optional SL/CP justifications and outlist |
| `cmd_retract` / `cmd_assert` | `retract` / `assert` | Changes truth value and prints cascade results |
| `cmd_what_if` | `what-if` | Read-only simulation of retract/assert with depth-grouped output |
| `cmd_explain` | `explain` | Prints the justification chain for why a node is IN or OUT |
| `cmd_derive` | `derive` | LLM-powered derivation — builds prompt, invokes model, parses proposals, optionally auto-applies |
| `cmd_review_beliefs` | `review-beliefs` | LLM-reviewed validity/sufficiency/necessity audit of derived beliefs |
| `cmd_contradictions` | `contradictions` | LLM-detected contradictions with optional auto-apply of nogoods |
| `cmd_ask` | `ask` | Natural language question answering over the belief network |
| `cmd_deduplicate` | `deduplicate` | Finds duplicate beliefs by ID-token or embedding similarity |
| `cmd_import_agent` / `cmd_sync_agent` | `import-agent` / `sync-agent` | Multi-agent federation: imports another agent's beliefs with namespacing |

### Output helpers

- **`_print_cascade(result)`** — Splits cascade into `went_out` / `went_in` lists with `[-]` / `[+]` markers.
- **`_print_restoration_hints(hints)`** — After retraction, prints advice when multi-premise SL nodes go OUT but some premises survive — suggests `--any` mode.
- **`_print_what_if_results(result, action, node_id)`** — Depth-grouped output for what-if simulations.
- **`_warn_multi_premise(premise_count, any_mode)`** — Prints a tip when an SL justification has 3+ premises and `--any` wasn't used.
- **`_parse_visible_to(args)`** — Parses the `--visible-to TAG,TAG` access control filter into a list.

### Derive internals

- **`_derive_one_round(args, ...)`** — Extracted function that runs a single derive round: loads network, builds prompt, invokes LLM, parses/validates proposals, applies or writes them. Returns count of beliefs added (0 = saturated, -1 = error).
- **`_write_derive_report(report_state, status)`** — Writes JSON reports to `reviews/` for audit trail.

## 3. Patterns

**Thin CLI / Fat API**: Every `cmd_*` function follows the same pattern:
1. Extract args and call `api.some_function(**_backend_kwargs(args))`
2. Handle exceptions (typically `KeyError`, `ValueError`) by printing to stderr and exiting with code 1
3. Format the returned dict for human reading

**Dual backend**: The `_backend_kwargs` / `_require_sqlite` pair implements a strategy pattern for SQLite vs PostgreSQL, selected by a single flag. Some commands gate on SQLite-only.

**Deferred imports**: Heavy dependencies (`derive`, `ask`, `cluster`, `llm`, `mcp_client`) are imported inside their respective `cmd_*` functions rather than at module level. This keeps `reasons status` fast even when ML libraries aren't installed.

**Plan-review-apply workflow**: Several commands (`derive`, `deduplicate`, `contradictions`) follow a three-phase pattern: (1) generate proposals to a file, (2) human reviews, (3) `--accept FILE` applies the reviewed plan. The `--auto` flag collapses phases 1-3.

**Exhaust mode**: `cmd_derive` supports `--exhaust` which loops `_derive_one_round` up to `--max-rounds` times, stopping when no new proposals are found (saturation).

## 4. Dependencies

**Imports (what it uses)**:
- `argparse`, `json`, `os`, `sys`, `pathlib.Path` — standard library
- `importlib.metadata.version` — for `--version`
- `reasons_lib.api` — **the only always-imported project module**; every command delegates here
- `reasons_lib.ask`, `reasons_lib.derive`, `reasons_lib.llm`, `reasons_lib.cluster`, `reasons_lib.mcp_client` — conditionally imported inside specific commands

**Imported by (what uses it)**:
- `tests/test_cli.py`, `tests/test_derive.py`, `tests/test_review.py`, `tests/test_contradictions.py`, `tests/test_ask.py`, `tests/test_ask_mcp.py`, `tests/test_pg_dispatch.py` — test modules that exercise CLI functions directly
- `reasons_lib/mcp_client.py` — likely for CLI entrypoint reference
- The massive `.venv/` list is noise from `importlib.metadata` resolution; ignore it

## 5. Flow

```
User types: reasons retract foo --reason "stale"
  → main()
  → argparse parses into args.command="retract", args.node_id="foo", args.reason="stale"
  → commands["retract"](args) → cmd_retract(args)
  → _backend_kwargs(args) → {"db_path": "reasons.db"}
  → api.retract_node("foo", reason="stale", db_path="reasons.db")
  → returns {"changed": ["bar", "baz"], "went_out": ["bar", "baz"], ...}
  → _print_cascade(result) → prints "[-] bar", "[-] baz"
```

For `derive --exhaust`:
```
cmd_derive → loop 1..max_rounds:
  _derive_one_round:
    api.export_network → full network snapshot
    build_prompt (from derive module) → LLM prompt string
    invoke_model (from llm module) → raw response text
    parse_proposals → list of proposal dicts
    validate_proposals → (valid, skipped)
    apply_proposals (if --auto/--exhaust) → adds to DB
    _write_derive_report → JSON to reviews/
  if added == 0: break (saturated)
```

## 6. Invariants

- **`_backend_kwargs` always returns either `{"db_path": ...}` or `{"pg_conninfo": ..., "project_id": ...}`** — never both, never empty. PostgreSQL requires `--project-id`; missing it causes `sys.exit(1)`.
- **Commands dispatch via exact string match** in the `commands` dict at the bottom of `main()`. A subcommand registered in argparse but missing from this dict would raise `KeyError`.
- **`--any` and `--sl` interaction**: `_warn_multi_premise` fires when SL has 3+ premises and `--any` was not passed, nudging users toward OR-semantics.
- **`--cluster` and `--sample` are mutually exclusive** in `cmd_derive` — enforced explicitly before work begins.
- **Report files** use ISO timestamps with colons stripped (`T105808` format) for filesystem safety.
- **Exit code 1** is used for all error paths (missing nodes, validation failures, stale sources).

## 7. Error Handling

Every `cmd_*` function catches specific exceptions from `api.*` calls:
- `KeyError` → "node not found" type errors
- `ValueError` → validation failures (duplicate IDs, invalid arguments)
- `FileExistsError` → `cmd_init` when DB already exists
- `FileNotFoundError` → import commands when belief files don't exist
- `PermissionError` → access tag violations in `show`, `explain`, `trace`
- `IndexError` → `cmd_remove_justification` when index is out of range

All errors print to `stderr` and call `sys.exit(1)`. No exceptions propagate to the caller — the CLI is a terminal boundary.

`cmd_check_stale` exits with code 1 when stale nodes are found, making it usable as a CI gate.

The `_derive_one_round` function returns `-1` on error, `0` on saturation, or a positive count — `cmd_derive` uses this to control the exhaust loop and exit status.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The business logic layer that every CLI command delegates to; understanding its return-value contracts is essential for maintaining the CLI
- [function] `reasons_lib/derive.py:build_prompt` — How the LLM derivation prompt is assembled from the belief network, including budget/sampling/clustering strategies
- [file] `reasons_lib/storage.py` — The persistence layer behind `api.py`; understanding the SQLite schema explains why some commands are SQLite-only
- [function] `reasons_lib/cli.py:_backend_kwargs` — The dual-backend routing mechanism; any new command must call this and handle both SQLite and PostgreSQL paths
- [general] `plan-review-apply-pattern` — The three-phase workflow (propose → review file → `--accept`) used by `derive`, `deduplicate`, and `contradictions`; understanding it is key to adding new LLM-driven commands

## Beliefs

- `cli-delegates-all-logic-to-api` — Every `cmd_*` function delegates to `reasons_lib.api` and contains no business logic, database access, or network manipulation
- `cli-backend-kwargs-controls-storage` — `_backend_kwargs` is the single chokepoint that determines whether a command runs against SQLite or PostgreSQL, and every command must pass its return value to `api.*`
- `cli-heavy-imports-are-deferred` — Modules `derive`, `ask`, `cluster`, `llm`, and `mcp_client` are imported inside their respective command handlers, not at module top level
- `cli-sqlite-only-commands-exist` — `derive`, `ask`, `review-beliefs`, `deduplicate`, and `contradictions` call `_require_sqlite` and will `sys.exit(1)` if `--pg` is set
- `cli-derive-returns-negative-one-on-error` — `_derive_one_round` returns -1 for errors, 0 for saturation, or positive for beliefs added; `cmd_derive` uses this to control the exhaust loop

