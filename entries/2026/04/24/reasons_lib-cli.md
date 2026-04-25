# File: reasons_lib/cli.py

**Date:** 2026-04-24
**Time:** 16:43

# `reasons_lib/cli.py` — CLI Entry Point

## Purpose

This file is the terminal interface to the Reasons belief tracking system. It owns exactly one responsibility: **translating between argparse and `reasons_lib.api`**. Every subcommand is a thin function that calls an API function, formats the returned dict for human consumption, and sets the exit code. It does not contain any domain logic — no propagation, no persistence, no graph traversal. That all lives in `api.py` and below.

The entry point is `main()`, which builds the argument parser and dispatches to the appropriate `cmd_*` function via a dictionary lookup.

## Key Components

### `main()` — Parser Construction and Dispatch

Builds a single `argparse.ArgumentParser` with ~30 subcommands. Each subcommand gets its own sub-parser with typed arguments. Dispatch is a flat dict mapping command names to handler functions — no class hierarchy, no plugin system. If no command is given, it prints help and exits 1.

### `cmd_*` Functions — One Per Subcommand

Each follows the same contract:
- Takes a single `args` namespace from argparse
- Calls one `api.*` function, passing through arguments
- Prints formatted output to stdout (data) or stderr (errors/diagnostics)
- Calls `sys.exit(1)` on failure, returns normally on success

Notable handlers:

- **`cmd_add`** / **`cmd_add_justification`**: Parse comma-separated `--sl`, `--unless`, `--access-tags` strings and pass them to the API. After success, call `_warn_multi_premise` to hint about `--any` mode when 3+ antecedents are used in AND mode.

- **`cmd_retract`** / **`cmd_assert`**: Mutate node truth values and print cascade results via `_print_cascade`. Retract also prints restoration hints when multi-premise justifications partially survive.

- **`cmd_what_if`**: Read-only simulation — calls `api.what_if_retract` or `api.what_if_assert`, formats depth-layered output showing what *would* change, and explicitly notes "database NOT modified."

- **`cmd_derive`** / **`_derive_one_round`**: The most complex handler. Shells out to an external LLM CLI (`claude` or `gemini`) via `asyncio.create_subprocess_exec`, feeds it a prompt built from the current belief network, parses proposals from the response, and either writes them to a file for review or auto-applies them. Supports `--exhaust` mode that loops until saturation.

- **`cmd_deduplicate`**: Two-phase workflow — without `--accept`, it finds duplicate clusters and writes a plan file; with `--accept`, it reads the plan and applies retractions.

- **`cmd_propagate`**: The only handler that bypasses `api.py` entirely, going directly to `Storage` → `Network.recompute_all()` → `Storage.save()`. This is a design inconsistency.

### Helper Functions

- **`_print_cascade(result)`**: Formats `went_out` / `went_in` lists from retraction/assertion results.
- **`_print_restoration_hints(hints)`**: Advises the user when a multi-premise node went OUT but some of its premises survived — suggests using `--any` to create fallback justifications.
- **`_print_what_if_results(result, action, node_id)`**: Shared formatter for what-if simulation output with depth separators.
- **`_parse_visible_to(args)`**: Extracts and splits the `--visible-to` comma-separated string into a list of tags, returning `None` if absent.
- **`_warn_multi_premise(premise_count, any_mode)`**: Prints a tip when an SL justification has 3+ premises and `--any` wasn't used.

## Patterns

**Thin CLI / Fat API**: The CLI never does domain work. It delegates to `api.py`, which returns plain dicts. This makes the API independently testable and reusable (e.g., from LangGraph or other agents) without importing argparse.

**Convention-based dispatch**: `main()` maps command strings to functions in a flat dict. No dynamic discovery, no decorators — just explicit registration. Easy to grep, easy to understand.

**Two-phase review workflow**: Both `derive` and `deduplicate` default to writing a proposal file that the user reviews, then a second invocation (`accept` / `--accept`) applies it. The `--auto` flag bypasses review.

**Stderr for diagnostics, stdout for data**: Commands like `cmd_derive` print progress and stats to stderr while reserving stdout for machine-readable output (proposals). Export commands (`cmd_export`, `cmd_export_markdown`) write only to stdout unless `--output` is specified.

## Dependencies

**Imports from the project:**
- `reasons_lib.api` — every command except `cmd_propagate`
- `reasons_lib.storage.Storage` — only `cmd_propagate` (direct DB access)
- `reasons_lib.derive` — only `cmd_derive`, `cmd_accept`, and `_derive_one_round` (lazy imports)

**External imports:** `argparse`, `json`, `sys`, `pathlib.Path`, plus lazy imports of `asyncio`, `os`, `shutil` inside `_derive_one_round`.

**What depends on it:** This file is the CLI entry point referenced by the `[project.scripts]` section in `pyproject.toml` (as the `reasons` command). Nothing else in the codebase imports it.

## Flow

1. User runs `reasons <command> [args]`
2. `main()` parses arguments, looks up the handler in the `commands` dict
3. Handler extracts arguments from the `args` namespace, calls `api.*`
4. API returns a dict; handler formats and prints it
5. On error: catches specific exceptions (`KeyError`, `ValueError`, `FileNotFoundError`, `PermissionError`), prints to stderr, exits 1

For `derive --exhaust`, the flow loops: `cmd_derive` calls `_derive_one_round` repeatedly until it returns 0 (saturated) or hits `--max-rounds`.

## Invariants

- **Every subcommand has a `--db` argument** inherited from the parent parser, defaulting to `api.DEFAULT_DB`. All database access flows through this path.
- **`--visible-to` filtering** is available on read commands (`status`, `show`, `explain`, `trace`, `export`, `compact`, `search`, `lookup`, `list`) and always parsed by `_parse_visible_to`. It returns `None` (no filtering) when the flag is absent.
- **`cmd_check_stale` exits 1 if any node is stale** — designed for CI pipeline use.
- **`_derive_one_round` returns -1 on error, 0 on saturation, >0 on success** — `cmd_derive` uses this to control the exhaust loop.

## Error Handling

Errors follow a consistent pattern: catch a specific exception type from the API layer, print the message to stderr, exit 1. The exception types are well-partitioned:

- `KeyError` — node not found
- `ValueError` — invalid input (bad justification, duplicate ID)
- `FileNotFoundError` — missing import file or database
- `FileExistsError` — database already exists (init without `--force`)
- `PermissionError` — access tag violation

The `_derive_one_round` function returns `-1` instead of exiting, letting `cmd_derive` decide whether to abort the exhaust loop or exit directly.

No exceptions are silently swallowed. Unhandled exceptions propagate as tracebacks, which is appropriate for a developer-facing CLI tool.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The functional API layer that every CLI command delegates to; understanding its return dict contracts is essential for modifying CLI output
- [function] `reasons_lib/derive.py:build_prompt` — How the belief network is serialized into an LLM prompt for the derive command
- [function] `reasons_lib/network.py:recompute_all` — The BFS propagation engine that `cmd_propagate` calls directly, bypassing the API layer
- [file] `reasons_lib/import_agent.py` — Agent import/sync logic that `cmd_import_agent` and `cmd_sync_agent` depend on for multi-agent belief federation
- [general] `two-phase-review-pattern` — The derive/accept and deduplicate/--accept workflows share a pattern worth understanding as a unit

## Beliefs

- `cli-is-pure-presentation` — Every `cmd_*` function except `cmd_propagate` delegates all domain logic to `api.py` and only formats output; `cmd_propagate` breaks this by calling `Storage` and `Network` directly
- `derive-uses-subprocess-not-sdk` — The derive command invokes LLMs by shelling out to `claude` or `gemini` CLI binaries via `asyncio.create_subprocess_exec`, not through any Python SDK
- `check-stale-exits-nonzero` — `cmd_check_stale` calls `sys.exit(1)` when stale nodes are found, making it usable as a CI gate
- `visible-to-is-optional-filter` — `_parse_visible_to` returns `None` when `--visible-to` is absent, which the API interprets as "no access restriction"; it never defaults to an empty list
- `derive-round-returns-negative-on-error` — `_derive_one_round` returns -1 on error, 0 on saturation, and a positive count on success, using return values instead of exceptions for flow control in exhaust mode

