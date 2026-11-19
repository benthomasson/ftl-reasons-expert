---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_example.py

## Purpose

`tests/test_example.py` verifies support for the `--example` feature on node creation and display. It checks both layers:

- **API layer**: `api.add_node(..., example=...)` persists the value into node metadata, and `api.show_node(...)` returns it.
- **CLI layer**: `reasons add ... --example ...` stores the example, and `reasons show ...` renders it.

The file’s docstring says it covers “add, update, and show”, but the actual tests only cover **add** and **show**. There are no update-command tests in this file, which is important coverage context.

## Key Components

### `run_cli(*args, db_path=None)`

A local test helper that invokes the CLI in-process.

Contract:

- Builds `argv` as `["reasons"]`.
- If `db_path` is provided, inserts `--db <db_path>` before the command: `["reasons", "--db", db, "add", ...]`.
- Appends the command arguments.
- Patches:
  - `sys.argv`
  - `sys.stdout` with `StringIO`
  - `sys.stderr` with `StringIO`
- Calls `reasons.cli.main()`.
- Catches `SystemExit` and converts it to an integer exit code:
  - `e.code if e.code is not None else 0`
- Returns `(stdout_text, stderr_text, exit_code)`.

This helper lets CLI tests assert on output and exit status without spawning subprocesses.

### `TestAddNodeExample`

API-level tests for `api.add_node`.

- `test_add_with_example`:
  - Adds node `n1` with `example="x = 1 + 2"`.
  - Asserts `api.add_node(...)` returns `node_id == "n1"`.
  - Asserts `api.show_node("n1", ...)["metadata"]["example"]` equals the original string.

- `test_add_without_example`:
  - Adds node without `example`.
  - Asserts the `"example"` key is absent from metadata, not merely empty or `None`.

- `test_add_example_with_access_tags`:
  - Adds node with both `example` and `access_tags`.
  - Confirms metadata preserves both fields independently.

### `TestCmdAddExample`

CLI-level tests for the `add` command.

- `test_cli_add_with_example`:
  - Runs `reasons add n1 "A belief" --example "reasons add n1 'A belief'"`.
  - Asserts exit code `0`.
  - Confirms the example value is persisted to metadata.

- `test_cli_add_without_example`:
  - Runs `add` without `--example`.
  - Asserts exit code `0` and absence of `"example"` in metadata.

### `TestCmdShowExample`

CLI-level tests for the `show` command.

- `test_show_displays_example`:
  - Seeds a node through the API with `example="x = func()"`.
  - Runs `reasons show n1`.
  - Asserts stdout contains `"Example:"` and the example content.

- `test_show_multiline_example`:
  - Seeds a three-line example:
    ```python
    x = 1
    y = 2
    print(x + y)
    ```
  - Runs `show`.
  - Parses stdout looking for a line whose stripped value starts with `"Example:"`.
  - After that, collects only lines starting with two spaces.
  - Skips blank lines.
  - Stops at the first non-indented, non-blank line.
  - Asserts exactly three captured example lines and that all begin with two spaces.

- `test_show_no_example`:
  - Seeds node without example.
  - Runs `show`.
  - Asserts `"Example:"` does not appear in stdout.

## Patterns

- **In-process CLI testing**: `run_cli` patches `sys.argv`, `sys.stdout`, and `sys.stderr`, then calls `main()` directly. This is faster and easier to assert against than subprocess invocation.
- **Two-layer testing**: The same feature is tested through the public API and through the CLI.
- **Temporary database isolation**: `tmp_path` creates a unique SQLite DB path per test, preventing cross-test state.
- **Metadata roundtrip assertions**: The tests verify values survive `add_node` → storage → `show_node` and `add` → storage → `show`.
- **Output parsing for formatting**: The multiline test does not just check content; it checks the visual contract that each example line is indented by two spaces.
- **Class-based grouping**: Test classes are grouped by feature area: API add, CLI add, CLI show.

## Dependencies

### Imports

- `sys`: patched for CLI argument simulation.
- `StringIO`: captures stdout/stderr.
- `unittest.mock.patch`: patches `sys.argv`, `sys.stdout`, `sys.stderr`.
- `pytest`: imported, though no direct `pytest.*` symbol is referenced. The `tmp_path` fixture is injected by pytest.
- `reasons.api`: used for `add_node` and `show_node`.
- `reasons.cli.main`: imported lazily inside `run_cli`.

### Depends on

- `api.add_node(...)` accepting an `example=` keyword and persisting it under `metadata["example"]`.
- `api.show_node(...)` returning a dict with a `"node_id"` and `"metadata"` field.
- `reasons.cli.main` supporting:
  - global `--db <path>` before the subcommand,
  - `add` with `--example`,
  - `show` rendering an `Example:` section.
- `tmp_path` fixture from pytest.

### Imported by

- Nothing in the application imports this file. Pytest discovers it as a test module.

## Flow

1. Each test creates a temporary DB path via `tmp_path`.
2. API tests directly call:
   - `api.add_node(...)`
   - `api.show_node(...)`
3. CLI tests call `run_cli(...)`, which:
   - constructs `argv`,
   - redirects stdout/stderr,
   - invokes `reasons.cli.main()`,
   - catches `SystemExit` and returns the exit code.
4. CLI `add` tests then reopen the DB through `api.show_node` to verify persistence.
5. CLI `show` tests seed data through the API, invoke `run_cli("show", ...)`, and inspect captured stdout.
6. The multiline test scans stdout line by line:
   - enters capture mode at `Example:`,
   - records indented example lines,
   - ignores blank lines,
   - exits capture mode at the next non-indented content.

## Invariants

- Adding a node with `example=...` stores the exact string at `metadata["example"]`.
- Adding a node without `example` must leave the `"example"` key absent from metadata.
- `example` and `access_tags` must coexist in metadata without interfering.
- CLI `add --example` must persist the value identically to the API path.
- CLI `show` must print an `Example:` section when metadata contains an example.
- CLI `show` must not print `Example:` when no example exists.
- Multiline examples must be rendered with every example line prefixed by two spaces.
- `run_cli` normalizes a normal `main()` return to exit code `0`.
- CLI tests expect the global `--db` option to appear before the subcommand.

## Error Handling

- `run_cli` catches only `SystemExit`. Non-`SystemExit` exceptions propagate and fail the test.
- If `SystemExit.code` is `None`, it is normalized to `0`.
- `stderr` is captured and returned but generally ignored by the assertions.
- The tests only cover successful command paths; they do not assert on error messages or nonzero exit behavior for invalid input.

## Topics to Explore

- [file] `reasons/cli.py` — Implements `add`, `show`, `--db`, and `--example` handling under test.
- [function] `reasons/api.py:add_node` — Defines how `example` and `access_tags` are stored in metadata.
- [function] `reasons/api.py:show_node` — Defines the node metadata shape returned to tests.
- [general] `example-update-coverage` — The docstring mentions update support, but this file contains no update tests; inspect whether update actually supports examples.
- [file] `tests/test_metadata.py` — Broader metadata behavior that may explain storage conventions used here.

## Beliefs

- `example-flag-persists-to-metadata` — CLI `add --example` stores the value under `metadata["example"]` and returns exit code 0.
- `omitted-example-omits-metadata-key` — Adding a node without `example` leaves the metadata without an `"example"` key.
- `show-renders-example-section` — The `show` command emits an `Example:` section when an example exists and omits it otherwise.
- `multiline-example-lines-are-indented` — Multiline examples are displayed with each example line prefixed by two spaces.
- `run-cli-converts-system-exit-to-code` — The `run_cli` helper catches `SystemExit` and returns its code, defaulting `None` to `0`.
