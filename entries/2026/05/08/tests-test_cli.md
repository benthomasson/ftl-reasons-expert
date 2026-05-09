# File: tests/test_cli.py

**Date:** 2026-05-08
**Time:** 14:08

## Purpose

`tests/test_cli.py` is the integration test suite for the `reasons` CLI. It exercises every user-facing subcommand by invoking `main()` with simulated `sys.argv`, capturing stdout/stderr, and asserting on exit codes and output strings. This is the primary safety net ensuring the CLI contract — argument parsing, command dispatch, output formatting, and error reporting — stays correct across changes.

It does **not** test the underlying library logic in isolation (that's handled by `test_storage.py`, `test_network.py`, etc.). Its responsibility is the CLI layer: does the right command produce the right output and exit code?

## Key Components

### `run_cli(*args, db_path=None)` (lines 14–25)

The test harness. Builds a synthetic `sys.argv`, patches `sys.stdout` and `sys.stderr` with `StringIO` buffers, calls `main()`, and returns `(stdout, stderr, exit_code)`. Catches `SystemExit` to extract the exit code rather than letting it abort the test process.

Contract: every CLI test goes through this function. It isolates tests from the real filesystem (via `tmp_path`-generated `db_path`) and from real stdio.

### `db_path` fixture (lines 12–13)

A pytest fixture yielding a path to a fresh SQLite DB in `tmp_path`. Each test gets its own database — no cross-test contamination.

### Test classes (~30 classes, ~90 tests)

Each class maps to a CLI subcommand or a cluster of related subcommands:

| Class | Subcommand(s) | What it validates |
|---|---|---|
| `TestInit` | `init`, `init --force` | DB creation, idempotency guard, force override |
| `TestAdd` | `add` | Premises, derived nodes (`--sl`), duplicates, access tags, multi-premise tip |
| `TestRetractAssert` | `retract`, `assert` | Retraction cascades, re-assertion cascades, idempotent operations, reason tracking |
| `TestStatus` | `status` | Empty DB, node listing, `--visible-to` filtering |
| `TestShow` | `show` | Node detail display, access control, dependents, retract reason, source/hash |
| `TestExplain` | `explain` | Premise vs. derived, outlist display, failed antecedents, violated outlists, labels |
| `TestTrace` / `TestTraceAccessTags` | `trace`, `trace-access-tags` | Dependency tracing, access tag propagation |
| `TestWhatIf` | `what-if retract/assert` | Dry-run simulations without modifying state |
| `TestExport` / `TestImportExportJson` | `export`, `export-markdown`, `import-json` | JSON roundtrip fidelity, markdown output |
| `TestCheckStale` / `TestHashSources` | `check-stale`, `hash-sources` | Source-file staleness detection, hash backfill, force re-hash |
| `TestDerive` | `derive --dry-run` | LLM derivation prompt preview |
| `TestImportAgent` / `TestImportBeliefs` | `import-agent`, `sync-agent`, `import-beliefs` | Multi-agent belief federation, sync with removals, duplicate handling |
| `TestDeduplicate` | `deduplicate`, `--auto`, `--accept` | Cluster detection, plan acceptance |
| `TestCmdUpdate` | `update` | Text/source mutation, missing-node error, no-flag error |

## Patterns

**Black-box CLI testing**: Tests treat `main()` as a black box. They never inspect internal state directly (with one exception — `TestPropagateWithChanges` manipulates `Storage` to create an inconsistent state). This makes the tests resilient to refactors of the CLI internals.

**Setup-by-CLI**: Rather than using fixtures or factories to seed the database, each test builds its state by calling `run_cli("add", ...)` sequentially. This ensures tests exercise the same code path users do, at the cost of slower tests and implicit coupling to `add`'s behavior.

**Exit-code-as-contract**: The suite consistently asserts `code == 0` for success and `code == 1` for errors. This is the primary behavioral contract — the CLI uses exit codes, not exceptions, to communicate outcomes.

**Output-string assertions**: Tests assert on human-readable substrings (`"Initialized"`, `"Added a [IN]"`, `"Error"`). This couples tests to the exact CLI output format, which is intentional — output changes are user-facing breaking changes.

**`monkeypatch.chdir`**: Used in `TestCheckStale` and `TestHashSources` where the CLI resolves source files relative to CWD.

## Dependencies

**Imports:**
- `reasons_lib.cli.main` — the single entry point under test
- `reasons_lib.storage.Storage` — used once in `TestPropagateWithChanges` to create an intentionally inconsistent DB state
- `pytest`, `unittest.mock.patch`, `io.StringIO`, `json`, `sys`, `pathlib.Path` — standard test infrastructure

**Imported by:** Nothing. This is a leaf test module.

## Flow

A typical test:

1. `run_cli("init", db_path=...)` — creates a fresh SQLite database
2. One or more `run_cli("add", ...)` calls — populates beliefs
3. `run_cli("<command-under-test>", ...)` — exercises the target command
4. Assertions on `(out, err, code)` — validates the contract

The `run_cli` helper patches `sys.argv` so `argparse` inside `main()` sees the synthetic arguments, patches stdio so output is captured, and catches `SystemExit` so the test process survives error paths.

## Invariants

1. **Every test gets a fresh DB** — the `db_path` fixture creates a unique path per test via `tmp_path`.
2. **`init` must precede other commands** — most tests call `run_cli("init", ...)` first. Some (like `TestCmdUpdate.test_update_text`) skip it, implying `add` auto-initializes.
3. **Exit code 0 = success, 1 = error** — enforced across all ~90 tests.
4. **Errors go to stderr, results go to stdout** — tests assert `"Error" in err` for failures and check `out` for success messages.
5. **`what-if` never modifies state** — `TestWhatIf` asserts `"NOT modified"` appears in output.

## Error Handling

The CLI uses `SystemExit(1)` for all error conditions. `run_cli` catches this and returns the exit code as the third tuple element. Tests validate errors by checking:

- `code == 1` (the operation failed)
- Specific error text in `stderr` (`"Error"`, `"not found"`, `"Access denied"`, `"--force"`, `"at least one"`)

The test suite itself does **not** test for unexpected exceptions — if `main()` raises something other than `SystemExit`, `run_cli` propagates it and the test fails with a traceback. This is intentional: unexpected exceptions are bugs.

## Topics to Explore

- [file] `reasons_lib/cli.py` — The production code under test; maps 1:1 to the test classes here
- [function] `reasons_lib/storage.py:Storage` — The persistence layer that every CLI command ultimately delegates to
- [file] `tests/conftest.py` — Shared fixtures that may affect test behavior (e.g., additional DB setup)
- [file] `reasons_lib/network.py` — The TMS network model — understanding nodes, justifications, and truth propagation is key to understanding what the CLI commands do
- [general] `outlist-propagation-gap` — The known issue where outlist nodes aren't tracked in the dependents index, which explains why `TestPropagateWithChanges` needs to manually corrupt state to test `propagate`

## Beliefs

- `cli-tests-use-black-box-pattern` — All CLI tests invoke `main()` through `run_cli()` with patched argv/stdio rather than calling internal functions directly, with one exception in `TestPropagateWithChanges`
- `cli-exit-code-contract-zero-or-one` — Every CLI command returns exit code 0 on success and 1 on error; no other exit codes are used across the entire test suite
- `cli-tests-no-shared-db-state` — Each test method gets an isolated SQLite database via `tmp_path`, ensuring no cross-test contamination
- `run-cli-catches-system-exit-only` — The `run_cli` harness catches `SystemExit` but lets all other exceptions propagate, meaning unexpected errors fail the test with a full traceback
- `cli-errors-route-to-stderr` — Error messages (access denied, not found, duplicate) are written to stderr while success output goes to stdout, validated by separate `out`/`err` assertions throughout

