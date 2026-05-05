# File: reasons_lib/cli.py

**Date:** 2026-05-05
**Time:** 15:14

## `reasons_lib/cli.py` — Explanation

### 1. Purpose

This is the CLI frontend for the Reason Maintenance System (RMS). It defines the `reasons` command-line tool — an argparse-based interface with ~35 subcommands. The file's single responsibility is **parsing arguments and formatting output**; all domain logic lives in `reasons_lib.api` and specialized modules (`derive`, `ask`, `llm`). The docstring says it plainly: "Thin wrappers around reasons_lib.api."

The entry point is `main()`, which is wired as a console script via `pyproject.toml`.

### 2. Key Components

**`main()`** — Builds the full argparse tree (~300 lines of argument definitions), dispatches to a handler via a `commands` dict keyed by subcommand name. Every subcommand maps to a `cmd_*` function.

**`cmd_*` functions** — Each follows the same contract:
- Accept a single `args` namespace from argparse
- Call one `api.*` function, passing parsed args
- Format the returned dict for terminal output (plain text, not JSON)
- Exit with code 1 on error, 0 on success

**Notable command handlers:**

| Handler | What it does |
|---|---|
| `cmd_add` | Creates a node (premise or derived). Supports `--any` mode, access tags, namespaces |
| `cmd_retract` / `cmd_assert` | Toggle truth values with cascade reporting |
| `cmd_what_if` | Dry-run simulation of retract/assert — prints depth-layered impact |
| `cmd_derive` | LLM-powered derivation of new beliefs. Has an `--exhaust` loop mode |
| `cmd_deduplicate` | Two-phase: generate plan, then `--accept` to apply |
| `cmd_ask` | Natural language Q&A over the belief network via LLM synthesis |
| `cmd_review_beliefs` | LLM-based validity/sufficiency/necessity review with optional auto-retract |

**Helper functions:**

- `_print_cascade(result)` — Formats went_out/went_in lists from retraction/assertion results
- `_print_restoration_hints(hints)` — Advises when multi-premise SL nodes go OUT with surviving premises
- `_warn_multi_premise(premise_count, any_mode)` — Tip when 3+ premises are AND-joined
- `_parse_visible_to(args)` — Extracts comma-separated access tags into a list (or `None`)
- `_derive_one_round(args, round_num)` — Single iteration of the derive pipeline; factored out so `cmd_derive` can loop for `--exhaust`
- `_print_what_if_results(result, action, node_id)` — Shared formatter for what-if retract and assert output

### 3. Patterns

**Thin CLI / Fat API separation.** Every `cmd_*` function is a pure adapter: parse → call API → print. No business logic in this file. This means the API layer is independently testable without subprocess invocation.

**Uniform error handling.** Each handler catches specific exceptions (`KeyError`, `ValueError`, `FileNotFoundError`, `PermissionError`) from the API, prints to stderr, and exits with code 1.

**Result-dict protocol.** The API functions return plain dicts, not domain objects. The CLI formats these dicts — it never accesses the database or network directly (with one exception: `cmd_trace` calls `api.show_node` in a loop to display premise details).

**Deferred imports.** Heavy modules (`derive`, `ask`, `llm`) are imported inside their command handlers, not at module level. This keeps `reasons status` fast even when LLM dependencies are slow to load.

**Two-phase workflows.** Both `derive` and `deduplicate` support a review workflow: generate proposals to a file, then accept with a second invocation (`--auto` bypasses the review step).

### 4. Dependencies

**Imports:**
- `argparse`, `json`, `sys`, `pathlib.Path` — stdlib
- `importlib.metadata.version` — for `--version`
- `reasons_lib.api` — all domain logic (always imported)
- `reasons_lib.ask` — deferred, only in `cmd_ask`
- `reasons_lib.derive` — deferred, only in `_derive_one_round` and `cmd_accept`
- `reasons_lib.llm` — deferred, only in `_derive_one_round`
- `subprocess` — deferred, only in `_derive_one_round` (for timeout handling)

**Imported by:** `tests/test_cli.py`, `tests/test_ask.py`, `tests/test_review.py` (the `.venv` entries in the "Imported By" list are false positives from a broad grep — they import their own `cli` modules, not this one).

### 5. Flow

```
main()
  ├── argparse.parse_args()
  ├── lookup subcommand in commands dict
  └── call cmd_*(args)
        ├── extract/transform args (e.g., split comma-separated strings)
        ├── call api.some_function(...)
        ├── format result dict → print to stdout
        └── on error: print to stderr, sys.exit(1)
```

For `cmd_derive` with `--exhaust`:
```
cmd_derive(args)
  └── for round in 1..max_rounds:
        ├── _derive_one_round(args, round_num)
        │     ├── api.export_network() — fresh each round
        │     ├── build_prompt() from derive module
        │     ├── invoke_model() from llm module
        │     ├── parse_proposals() + validate_proposals()
        │     └── apply_proposals() if --auto/--exhaust
        └── break if added == 0 (saturated) or < 0 (error)
```

### 6. Invariants

- **Every subcommand must be registered in both** the argparse subparser tree and the `commands` dict in `main()`. If you add a new command and forget either, it silently fails.
- **`--db` defaults to `api.DEFAULT_DB`** — all handlers pass `args.db` through, so the database path is user-overridable on every command.
- **`_parse_visible_to` returns `None` (not empty list)** when `--visible-to` is absent — the API layer interprets `None` as "no access control."
- **`cmd_derive`'s `--exhaust` implies `--auto`** — the `_derive_one_round` function checks `args.auto or args.exhaust` before auto-applying proposals.
- **`_derive_one_round` returns an int:** positive = beliefs added, 0 = saturated, negative = error. This is the loop control signal for `--exhaust`.

### 7. Error Handling

All errors follow one pattern:
```python
try:
    result = api.some_function(...)
except (KeyError, ValueError, ...) as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

Exceptions are caught at the boundary — the API raises, the CLI catches and exits. No exceptions propagate to the user as tracebacks. The one special case is `cmd_init`, which catches `FileExistsError` and suggests `--force`.

`cmd_check_stale` exits with code 1 when stale nodes are found (even though it's not an "error" per se) — this makes it usable as a CI gate.

`_derive_one_round` has broader exception handling: it catches `FileNotFoundError` (model binary missing), `subprocess.TimeoutExpired`, and a generic `Exception` fallback for model invocation failures.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The domain logic layer that every `cmd_*` function delegates to; understanding the result-dict contracts is essential
- [function] `reasons_lib/derive.py:build_prompt` — How the LLM derivation prompt is constructed from network state, including budget/sampling/depth filters
- [file] `reasons_lib/ask.py` — The `ask` function implements FTS5 search + LLM synthesis with multiple modes (simple, dual, natural)
- [function] `reasons_lib/cli.py:_derive_one_round` — The most complex handler in this file; orchestrates export → prompt → model → parse → validate → apply
- [general] `two-phase-review-workflow` — How `derive` and `deduplicate` generate proposal files for human review before applying changes

---

## Beliefs

- `cli-is-pure-adapter` — Every `cmd_*` function calls exactly one `api.*` function and formats its result; no business logic lives in cli.py (with the minor exception of `cmd_trace` calling `api.show_node` in a display loop)
- `derive-one-round-return-protocol` — `_derive_one_round` returns a positive int (beliefs added), 0 (saturated/no proposals), or negative (error); this is the contract `cmd_derive`'s exhaust loop depends on
- `deferred-imports-for-llm-modules` — `ask`, `derive`, `llm`, and `subprocess` are imported inside command handlers, not at module level, so non-LLM commands stay fast
- `check-stale-exits-nonzero-on-stale` — `cmd_check_stale` calls `sys.exit(1)` when stale nodes exist, making it usable as a CI/pre-commit gate
- `commands-dict-must-mirror-subparsers` — Adding a subcommand requires entries in both the argparse subparser definitions and the `commands` dispatch dict in `main()`; omitting either silently breaks the command

