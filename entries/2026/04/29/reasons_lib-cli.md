# File: reasons_lib/cli.py

**Date:** 2026-04-29
**Time:** 16:58

## Purpose

`cli.py` is the terminal interface for the `reasons` command — a Truth Maintenance System (TMS). It owns exactly one responsibility: **parsing CLI arguments and formatting API results for human consumption**. Every command handler is a thin wrapper that calls a function in `reasons_lib.api`, then prints the returned dict in a readable format. The file never touches the database or network directly (with one exception: `cmd_propagate`).

## Key Components

### `main()` — Argument Parser & Dispatch

The bottom of the file builds an `argparse` tree with ~35 subcommands, then dispatches via a `commands` dict mapping subcommand names to `cmd_*` functions. No subclass or plugin system — just a flat dictionary lookup.

### Command Handlers (`cmd_*`)

Every handler follows the same contract:

1. Extract args from the `argparse.Namespace`
2. Call `api.<something>()` — passing `db_path=args.db` for database location
3. Print results to stdout (success) or stderr (errors)
4. Call `sys.exit(1)` on failure

Notable handlers:

- **`cmd_add` / `cmd_add_justification`** — Create nodes or add justifications. Both call `_warn_multi_premise` to hint about `--any` mode when 3+ premises form an AND conjunction.
- **`cmd_retract` / `cmd_assert`** — Toggle truth values with cascading side effects. Both use `_print_cascade` to show what went IN/OUT.
- **`cmd_what_if`** — Read-only simulation via `api.what_if_retract` / `api.what_if_assert`, formatted by `_print_what_if_results`. Never modifies the database.
- **`cmd_derive`** — The most complex handler. Shells out to an external CLI (`claude` or `gemini`) via `asyncio.create_subprocess_exec`, feeds it a prompt built from the current belief network, parses LLM-proposed beliefs, validates them, and optionally auto-applies. Supports `--exhaust` mode that loops up to `--max-rounds` times until saturation.
- **`cmd_deduplicate`** — Two-phase workflow: generates a dedup plan file, or applies one via `--accept`.

### Helper Functions

- **`_print_cascade(result)`** — Formats went-out/went-in lists from retraction or assertion cascades.
- **`_print_restoration_hints(hints)`** — When a multi-premise SL node goes OUT, tells the user which premises survived and how to restore with `--any`.
- **`_print_what_if_results(result, action, node_id)`** — Shared formatter for what-if simulations, showing depth-stratified impact.
- **`_parse_visible_to(args)`** — Extracts and splits the `--visible-to` comma-separated access tags into a list (or `None`).
- **`_warn_multi_premise(premise_count, any_mode)`** — Prints a tip when an SL justification has 3+ premises and `--any` wasn't used.
- **`_derive_one_round(args, round_num)`** — Factored out of `cmd_derive` to support both single-round and exhaust-loop modes.

## Patterns

**Thin CLI / Fat API separation.** The file's docstring states the design intent explicitly: "Thin wrappers around `reasons_lib.api`." The API returns dicts; the CLI formats them. This makes the API independently testable without argparse.

**Convention-based dispatch.** The `commands` dict at the end of `main()` maps subcommand strings to handler functions. Adding a new command means: write `cmd_foo`, add an argparse subparser, add one dict entry.

**Consistent error handling idiom.** Nearly every handler wraps the API call in `try/except`, prints to stderr, exits with code 1. The exception types caught are specific to each API call (`KeyError` for missing nodes, `ValueError` for invalid input, `PermissionError` for access control).

**Lazy imports.** Heavy modules (`asyncio`, `shutil`, `derive`, `ask`, `Storage`) are imported inside function bodies, not at module level. This keeps `reasons --help` fast.

## Dependencies

**Inbound:**
- `reasons_lib.api` — the real logic layer; every `cmd_*` delegates here
- `reasons_lib.storage.Storage` — used directly only in `cmd_propagate` (the one handler that bypasses the API)
- `reasons_lib.ask.ask` — lazy-imported for the `ask` subcommand
- `reasons_lib.derive` — lazy-imported for `derive` and `accept` subcommands
- Standard library: `argparse`, `json`, `sys`, `pathlib`, `asyncio`, `os`, `shutil`, `importlib.metadata`

**Outbound:**
- `tests/test_cli.py` — tests the CLI handlers
- `tests/test_ask.py` — imports for testing the ask integration
- Entry point: registered in `pyproject.toml` as the `reasons` console script

## Flow

1. User runs `reasons <subcommand> [args]`
2. `main()` parses args, looks up handler in `commands` dict
3. Handler calls `api.*()` with parsed arguments
4. API returns a result dict
5. Handler formats and prints the dict
6. On error: prints to stderr, exits 1

The `derive` flow is different — it builds a prompt from the exported network, invokes an external LLM CLI as a subprocess, parses the LLM's markdown response into structured proposals, validates them against the current network, then either writes a proposals file or auto-applies them.

## Invariants

- **`--db` defaults to `api.DEFAULT_DB`** — every handler threads `args.db` through to the API, so the database path is always explicit.
- **`cmd_check_stale` exits 1 when stale nodes exist** — this is designed for CI: a non-zero exit signals the network is out of sync with source files.
- **`_derive_one_round` returns -1 on error, 0 on saturation, positive on success** — `cmd_derive` uses this to decide whether to continue the exhaust loop or abort.
- **`what-if` never modifies the database** — the output explicitly says "(database NOT modified)".
- **`cmd_propagate` is the only handler that bypasses the API** — it directly instantiates `Storage`, loads the network, calls `recompute_all()`, and saves. This is a design smell noted as a known inconsistency.

## Error Handling

Errors follow a strict pattern: catch specific exceptions from the API, print to stderr, exit 1. The API raises:
- `KeyError` — node not found
- `ValueError` — invalid input (bad justification, duplicate ID)
- `FileExistsError` — database already exists (in `init` without `--force`)
- `PermissionError` — access tag violation
- `FileNotFoundError` — missing import files

The `derive` path also handles `TimeoutError` from the subprocess and generic `Exception` from model invocation failures. Errors in derive return -1 to the caller rather than exiting directly, so the exhaust loop can report which round failed.

## Topics to Explore

- [file] `reasons_lib/api.py` — The real logic layer; understanding the dicts it returns is essential to understanding what the CLI formats
- [function] `reasons_lib/derive.py:build_prompt` — How the belief network gets serialized into an LLM prompt for automated derivation
- [function] `reasons_lib/network.py:recompute_all` — The truth maintenance propagation algorithm that `cmd_propagate` calls directly
- [file] `tests/test_cli.py` — Shows how the CLI handlers are tested and what edge cases are covered
- [general] `outlist-gate-evaluation` — The known issue where GATE beliefs aren't auto-re-evaluated when outlist nodes change; `cmd_propagate` is the manual workaround

## Beliefs

- `cli-delegates-all-logic-to-api` — Every `cmd_*` handler delegates to `reasons_lib.api` and only formats the returned dict, with the sole exception of `cmd_propagate` which directly uses `Storage`
- `cli-exit-1-on-error` — All error paths print to stderr and call `sys.exit(1)`; no handler silently swallows exceptions
- `derive-returns-negative-one-on-error` — `_derive_one_round` returns -1 on error, 0 on saturation, and a positive int for the number of beliefs added, which `cmd_derive` uses to control the exhaust loop
- `check-stale-exits-nonzero` — `cmd_check_stale` exits with code 1 when any stale nodes are found, making it usable as a CI gate
- `cli-uses-lazy-imports-for-heavy-modules` — `asyncio`, `derive`, `ask`, and `Storage` are imported inside function bodies to keep startup fast

