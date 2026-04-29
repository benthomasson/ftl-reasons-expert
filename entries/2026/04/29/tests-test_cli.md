# File: tests/test_cli.py

**Date:** 2026-04-29
**Time:** 17:10

## Purpose

`tests/test_cli.py` is the **integration test suite for the `reasons` CLI**. It tests every subcommand exposed by `reasons_lib.cli:main()` by invoking the full argument-parsing and dispatch pipeline — not by calling internal APIs directly. This makes it a contract test for the CLI's user-facing behavior: exit codes, stdout messages, and stderr error output.

It owns the responsibility of verifying that CLI commands produce the correct side effects on the SQLite belief store and return well-structured output. It is the most comprehensive test file in the project, covering ~40 subcommands across 35 test classes.

## Key Components

### `run_cli(*args, db_path=None)` — The Test Harness

The central helper that every test calls. It:

1. Constructs a synthetic `sys.argv` from the arguments (prepending `"reasons"` and optionally `--db <path>`)
2. Patches `sys.stdout` and `sys.stderr` with `StringIO` buffers to capture output
3. Calls `main()` and catches `SystemExit` to extract the exit code
4. Returns the tuple `(stdout, stderr, exit_code)`

This avoids subprocess overhead while still exercising the real argument parser and dispatch logic. The `SystemExit` catch is critical because `argparse` raises it on parse errors and the CLI raises it explicitly for error conditions.

### `db_path` Fixture

A pytest fixture providing a fresh temp-directory-based SQLite path for each test. Every test that modifies state initializes a new database with `run_cli("init", db_path=db_path)`, ensuring complete isolation.

### Test Classes (grouped by subcommand)

| Class | Commands Tested | Key Assertions |
|---|---|---|
| `TestInit` | `init`, `init --force` | Creates DB; refuses re-init without `--force` |
| `TestAdd` | `add`, `add --sl`, `add --access-tags` | Premises vs derived nodes; duplicate rejection; multi-premise tip |
| `TestAddJustification` | `add-justification` | Adding justifications to existing nodes |
| `TestRetractAssert` | `retract`, `assert` | Truth-value flipping; cascade propagation; idempotency ("already OUT/IN") |
| `TestStatus` | `status`, `status --visible-to` | Node listing; access-tag filtering |
| `TestShow` | `show` | Detail display; access denial; dependents listing; retract reasons; source hashes |
| `TestExplain` / `TestExplainBranches` | `explain` | Justification tree rendering including outlists, failed antecedents, violated outlists, labels |
| `TestTrace` | `trace` | Recursive premise tracing |
| `TestWhatIf` / `TestWhatIfEdgeCases` | `what-if retract`, `what-if assert` | Dry-run simulation without modifying the store |
| `TestExport` / `TestImportExportJson` | `export`, `import-json` | JSON roundtrip fidelity |
| `TestImportBeliefs` / `TestImportAgent` | `import-beliefs`, `import-agent`, `sync-agent` | Markdown belief import; agent namespace operations; sync with removals |
| `TestCheckStale` / `TestHashSources` | `check-stale`, `hash-sources` | Source-file hash tracking; staleness detection on modification and deletion |
| `TestDerive` | `derive --dry-run` | LLM derivation prompt generation (dry-run only) |
| `TestChallenge` / `TestNogood` | `challenge`, `defend`, `nogood` | Dialectical operations; contradiction recording |
| `TestDeduplicate` | `deduplicate`, `deduplicate --auto`, `deduplicate --accept` | Duplicate detection and resolution |

## Patterns

**Black-box CLI testing**: Tests never import internal modules (with two exceptions — `Storage` in `TestPropagateWithChanges` to artificially corrupt state, and `Path` for file I/O). Everything goes through `run_cli → main()`.

**Setup-per-test via chained CLI calls**: Each test builds up its own belief network by sequentially calling `run_cli("init", ...)`, then `run_cli("add", ...)`, etc. This is deliberate — it tests the CLI commands as the user would chain them, not synthetic database states.

**Convention: exit code 0 = success, 1 = error**: Every test asserts `code == 0` for happy paths and `code == 1` for errors. Error details are always on stderr; success output on stdout.

**Assertion on human-readable strings**: Tests assert on substrings of output (`"Added a [IN]"`, `"Went OUT"`, `"already IN"`). This couples tests to UI copy, which is intentional — the CLI's output **is** its API for human consumers.

**`monkeypatch.chdir`** for source-file tests: `TestCheckStale` and `TestHashSources` use `monkeypatch.chdir(tmp_path)` because `check-stale` resolves source paths relative to the working directory.

## Dependencies

**Imports**:
- `reasons_lib.cli:main` — the only production import (the entire CLI entry point)
- `reasons_lib.storage:Storage` — used once to corrupt DB state for `TestPropagateWithChanges`
- `pytest`, `json`, `sys`, `io.StringIO`, `unittest.mock.patch`, `pathlib.Path` — standard test infrastructure

**Imported by**: Nothing. This is a leaf test module.

## Flow

A typical test follows this pattern:

```
init DB → build belief network → execute target command → assert exit code + output
```

For example, `test_retract_cascade`:
1. `init` creates an empty SQLite DB
2. `add a "A"` creates premise `a` (IN)
3. `add b "B" --sl a` creates derived node `b` depending on `a` (IN)
4. `retract a` retracts `a`, which cascades to `b`
5. Asserts: exit code 0, output contains `"Went OUT"` and `"b"`

The `what-if` tests verify that the same cascade logic runs but **does not persist** — asserting `"NOT modified"` in output.

## Invariants

1. **Every test starts with `run_cli("init")`** — no test assumes a pre-existing database.
2. **Exit code semantics are strict**: 0 means the operation succeeded; 1 means a user-facing error occurred (missing node, access denied, file not found). No test expects any other exit code.
3. **Error messages go to stderr, success messages go to stdout** — tests assert on the correct stream.
4. **Access-tag filtering is enforced at the CLI layer**: `--visible-to` filters appear on `status`, `show`, `explain`, `trace`, and `trace-access-tags`, and each is tested for both allowed and denied access.
5. **Idempotency guards**: `retract` on an already-OUT node and `assert` on an already-IN node both return code 0 with informational messages, not errors.

## Error Handling

The `run_cli` harness catches `SystemExit` and returns its code, preventing tests from crashing pytest. All CLI errors are surfaced as:
- `sys.exit(1)` with a message on stderr containing `"Error"` or a specific diagnostic string
- Tests validate both the exit code **and** the error message content

Edge cases are well-covered:
- Missing nodes (`"Error"` in stderr, code 1)
- Missing files for import commands (code 1)
- Access-denied for tagged beliefs (`"Access denied"` in stderr)
- Duplicate node IDs (`"Error"` in stderr)
- Re-initialization without `--force` (`"--force"` hint in stderr)

---

## Topics to Explore

- [file] `reasons_lib/cli.py` — The production CLI dispatch: argument parsing, subcommand handlers, and exit-code conventions that these tests validate
- [function] `reasons_lib/storage.py:Storage` — The SQLite persistence layer that every CLI command reads/writes through — understanding it explains why `db_path` isolation works
- [file] `tests/conftest.py` — Shared pytest fixtures that may affect test behavior (e.g., whether `db_path` is duplicated or extended there)
- [general] `outlist-cascade-semantics` — How `--unless` (outlist) interacts with `retract`/`assert` cascades — tested in `TestExplainBranches` but the logic lives in the network module
- [file] `reasons_lib/network.py` — The in-memory belief network and propagation engine; the core data structure behind every CLI operation

## Beliefs

- `cli-tests-are-black-box-integration` — All tests invoke `main()` through the full argv-parsing pipeline rather than calling internal APIs, with one exception (`TestPropagateWithChanges` directly mutates storage to create inconsistent state)
- `cli-exit-code-contract-is-binary` — Every CLI subcommand returns exit code 0 for success and 1 for any user-facing error; no other codes are tested or expected
- `cli-errors-use-stderr-success-uses-stdout` — Error diagnostics are written to stderr and success output to stdout; tests consistently assert on the correct stream
- `each-test-creates-isolated-db` — Every test method initializes a fresh SQLite database via `run_cli("init")` in a pytest `tmp_path`, ensuring zero shared state between tests
- `run-cli-helper-catches-systemexit` — The `run_cli` harness intercepts `SystemExit` to extract exit codes, preventing argparse errors or explicit `sys.exit()` calls from terminating the test process

