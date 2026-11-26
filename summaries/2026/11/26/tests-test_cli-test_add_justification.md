---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: test_add_justification in tests/test_cli.py

## Overview

`test_add_justification` is a pytest test method in `tests/test_cli.py`, inside `TestAddJustification`. It is an end-to-end CLI smoke test for the `add-justification` command: it sets up a temporary database, creates two nodes, adds a justification from one node to another, and asserts that the command succeeds and prints the expected success message.

## Purpose

The test exists to verify the happy path of the CLI command:

```bash
reasons --db <db_path> add-justification b --sl a
```

Specifically, it checks that:

1. `add-justification` is wired into `main()`.
2. It accepts an existing target node and a support-link node via `--sl`.
3. It exits with code `0` on success.
4. It prints `"Added justification to b"` to stdout.

It is not a unit test of the underlying justification logic. It is a CLI integration test using `run_cli()` and the real `reasons.cli.main` entry point.

## Contract

### Preconditions

- pytest supplies a valid `db_path` via the module-level `db_path` fixture:
  ```python
  @pytest.fixture
  def db_path(tmp_path):
      return str(tmp_path / "test.db")
  ```
- The parent directory exists and is writable.
- The database file does not need to exist initially; `init` creates it.
- The `reasons.cli.main` entry point is importable and supports the `init`, `add`, and `add-justification` subcommands.
- The CLI accepts the global `--db` option before the subcommand, because `run_cli()` builds argv as:
  ```python
  ["reasons", "--db", db_path, *args]
  ```

### Postconditions

- A temporary SQLite database exists at `db_path`.
- Nodes `a` and `b` have been added.
- The command `add-justification b --sl a` exits successfully.
- Captured stdout contains the substring `"Added justification to b"`.
- The test itself returns `None`; pytest reports pass/fail based on assertions.

### Invariants

- Each `run_cli()` call patches `sys.argv`, `sys.stdout`, and `sys.stderr` and restores them after the call.
- The database state persists across the sequential `run_cli()` calls because all calls use the same `db_path`.
- The test does not inspect stderr.
- The test does not inspect the database after `add-justification`.
- The test does not assert that the justification actually references `a`, nor that `b`’s truth value or classification changed.

### Assumptions Not Enforced by Types

- The exact stdout message is `"Added justification to b"`.
- `--sl` means support link / antecedent for the new justification.
- A single node ID is valid for `--sl`.
- `add-justification` uses a nonzero `SystemExit` code for failure, and normal return / `SystemExit(0)` for success.
- If the CLI handler were to print an error but return normally, `run_cli()` would still report code `0`; this test would only catch that if the success substring were missing.

## Parameters

### `self`

The pytest test instance. Not used directly.

### `db_path`

A string path to a temporary SQLite database file, provided by the `db_path` fixture. It is passed to every `run_cli()` call via the global `--db` option.

Expected characteristics:

- Type: `str`
- Writable path
- Parent directory exists
- Initially may not exist
- Should be unique per test because `tmp_path` is function-scoped by default

## Return Value

The test method itself returns `None`. Its “result” is pytest pass/fail based on assertions.

The helper `run_cli()` returns a tuple:

```python
(out, err, code)
```

- `out`: captured stdout as a string
- `err`: captured stderr as a string
- `code`: `e.code` if `main()` raises `SystemExit`, otherwise `0` if `main()` returns normally

This test consumes only `out` and `code` from the `add-justification` invocation.

## Algorithm

1. Initialize a fresh database:
   ```python
   run_cli("init", db_path=db_path)
   ```

2. Add node `a` with text `"A"`:
   ```python
   run_cli("add", "a", "A", db_path=db_path)
   ```

3. Add node `b` with text `"B"`:
   ```python
   run_cli("add", "b", "B", db_path=db_path)
   ```

4. Run the command under test:
   ```python
   out, err, code = run_cli("add-justification", "b", "--sl", "a", db_path=db_path)
   ```
   Internally, `run_cli()` builds argv equivalent to:
   ```python
   ["reasons", "--db", db_path, "add-justification", "b", "--sl", "a"]
   ```
   It patches `sys.argv`, `sys.stdout`, and `sys.stderr`, then calls `main()`.

5. Assert success exit code:
   ```python
   assert code == 0
   ```

6. Assert expected success output:
   ```python
   assert "Added justification to b" in out
   ```

No check is made against `err`, and no database state is inspected after the command.

## Side Effects

- Creates and mutates a temporary SQLite database at `db_path`.
- Adds nodes `a` and `b` before the command under test.
- Invokes the real CLI dispatcher and command handler via `main()`.
- Captures stdout/stderr instead of writing to the real terminal.
- Temporarily replaces `sys.argv`, `sys.stdout`, and `sys.stderr` during each `run_cli()` call; these are restored by context managers afterward.
- Leaves the temporary database under pytest’s `tmp_path`, which pytest normally cleans up.
- The `add-justification` command may update the database by adding a justification to node `b` and may trigger internal re-evaluation/propagation logic, but this test does not assert those effects.

## Error Handling

- `run_cli()` catches `SystemExit` and returns its `code`.
- If `main()` returns normally, `run_cli()` returns code `0`.
- If `main()` raises a non-`SystemExit` exception, it propagates and the test fails.
- If `add-justification` exits nonzero, `assert code == 0` fails.
- If stdout does not contain the exact substring `"Added justification to b"`, the test fails.
- The initial `init` and `add` calls are not asserted. If they fail, later assertions may fail indirectly, but the test does not explicitly validate their exit codes.
- A nonempty stderr is ignored by this test.

## Usage Patterns

This is a pytest-discovered test method. It should be run as part of the `TestAddJustification` class, not called directly.

Typical invocation:

```bash
pytest tests/test_cli.py::TestAddJustification::test_add_justification
```

Caller obligations:

- Ensure the `db_path` fixture is available.
- Ensure `reasons.cli.main` is importable.
- Ensure the CLI subcommands and option names used here remain stable.
- Use `run_cli()` for CLI-level tests so stdout/stderr and `SystemExit` are captured.

## Dependencies

- `pytest` for test discovery and fixtures
- `tmp_path` pytest fixture via `db_path`
- `run_cli()` helper from the same file
- `reasons.cli.main`
- `sys`, `io.StringIO`, and `unittest.mock.patch` inside `run_cli()`
- Underlying CLI command handlers and storage layer used by `init`, `add`, and `add-justification`

## Topics to Explore

- [file] `tests/test_add_justification.py` — Covers add-justification behavior beyond this CLI happy-path smoke test.
- [function] `tests/test_cli.py:run_cli` — Shows exactly how CLI argv, stdout/stderr, and exit codes are mocked and captured.
- [function] `reasons/cli.py:main` — Entry point that parses the `add-justification` command and dispatches to its handler.
- [general] `add-justification-cascade` — Explains cascade/re-evaluation behavior when adding a justification changes node truth values.
- [file] `reasons/cli.py` — Contains the actual CLI parser, `--sl` handling, and success-message formatting for `add-justification`.

## Beliefs

- `test-add-justification-is-a-cli-integration-test` — It calls `main()` through `run_cli()` instead of invoking an `add-justification` handler directly.
- `add-justification-requires-existing-target-and-support-node` — The test creates nodes `a` and `b` before running `add-justification b --sl a`.
- `run-cli-catches-systemexit-and-returns-exit-code` — `run_cli()` captures `SystemExit.code` and returns `0` when `main()` returns normally.
- `test-add-justification-asserts-only-exit-code-and-stdout-substring` — The test checks `code == 0` and `"Added justification to b" in out`, not stderr or persisted database state.
- `run-cli-assumes-db-option-precedes-subcommand` — `run_cli()` builds argv as `reasons --db <path> <subcommand> ...`, so the CLI must accept `--db` before the subcommand.
