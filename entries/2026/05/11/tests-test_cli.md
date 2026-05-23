# File: tests/test_cli.py

**Date:** 2026-05-11
**Time:** 12:58

## Purpose

`test_cli.py` is the integration test suite for the `reasons` CLI — the command-line interface to a Truth Maintenance System (TMS). It tests every CLI subcommand by invoking `main()` with synthetic `sys.argv`, capturing stdout/stderr, and asserting on exit codes and output strings. This file owns the contract between the user-facing CLI and the underlying library: if a command's output format, error messaging, or exit code changes, these tests break.

It is not a unit test file — it exercises the full stack from argument parsing through storage, network propagation, and output formatting, using real SQLite databases in `tmp_path`.

## Key Components

### `run_cli(*args, db_path=None)` (lines 14–24)

The central test harness. Constructs a `sys.argv` list, patches `sys.stdout` and `sys.stderr` with `StringIO` buffers, calls `main()`, and returns a `(stdout, stderr, exit_code)` tuple. Catches `SystemExit` to extract the exit code — this is how argparse and the CLI signal success/failure. Every test in the file flows through this function.

Contract: returns `(str, str, int)`. Exit code `0` means success; `1` means error. The `db_path` kwarg injects `--db <path>` before the subcommand args, isolating each test to its own database.

### `db_path` fixture (lines 11–12)

A pytest fixture that returns a path to a nonexistent `.db` file inside `tmp_path`. Each test gets a fresh database location — the `init` subcommand creates it.

### Test classes

There are **35 test classes**, each covering one CLI subcommand or a cross-cutting concern. The naming convention is `Test<SubcommandName>` (e.g., `TestAdd`, `TestRetractAssert`, `TestWhatIf`). Within each class, methods follow the pattern `test_<scenario>` — happy path first, then error cases and edge cases.

Major groups:

| Class | What it covers |
|---|---|
| `TestInit` | Database creation, idempotency guard, `--force` |
| `TestAdd` | Premise and derived node creation, duplicates, access tags, multi-premise tip |
| `TestRetractAssert` | Retraction/assertion, cascades, idempotency, reasons |
| `TestStatus` / `TestShow` / `TestExplain` / `TestTrace` | Read-only inspection commands |
| `TestWhatIf` | Hypothetical retract/assert simulation |
| `TestExport` / `TestImportExportJson` / `TestImportBeliefs` / `TestImportAgent` | Serialization round-trips |
| `TestCheckStale` / `TestHashSources` | Source-file change detection |
| `TestDerive` | LLM-driven derivation (dry-run only) |
| `TestDeduplicate` / `TestDeduplicateSemantic` | Duplicate belief detection (text and semantic) |
| `TestCmdUpdate` | In-place node text/source editing |

## Patterns

**Black-box CLI testing via `main()` patching.** Rather than importing individual command handlers, every test patches `sys.argv` and calls the single `main()` entrypoint. This guarantees the argparse wiring is tested alongside the logic.

**Setup-by-command.** Tests don't use a shared fixture with pre-populated data. Instead, each test calls `run_cli("init", ...)` then `run_cli("add", ...)` to build its own state. This makes tests self-contained but verbose — a 4-node graph requires 5 `run_cli` calls before the assertion.

**Exit code as error signal.** Success is `code == 0`; errors are `code == 1`. Tests assert on the exit code first, then check stderr for error messages or stdout for success messages.

**String-matching assertions.** Tests verify output by checking for substring presence (`assert "Added a [IN]" in out`), not by parsing structured output. This is intentional — the CLI is human-facing, and the tests verify the user experience.

**Conditional skip for optional dependencies.** `TestDeduplicateSemantic` is guarded by `@pytest.mark.skipif(not HAS_CLUSTER_DEPS, ...)`, since semantic deduplication requires `sentence-transformers` and `scikit-learn`.

## Dependencies

**Imports:**
- `reasons_lib.cli.main` — the sole production import; the entire CLI surface
- `reasons_lib.storage.Storage` — used in one test (`TestPropagateWithChanges`) to directly corrupt database state
- `reasons_lib.cluster.HAS_CLUSTER_DEPS` — feature flag for optional semantic clustering
- Standard library: `json`, `sys`, `io.StringIO`, `unittest.mock.patch`, `pathlib.Path`
- `pytest` — fixtures, markers

**Imported by:** Nothing. This is a leaf test file.

## Flow

A typical test execution:

1. pytest calls the test method, injecting the `db_path` fixture (a temp file path).
2. The test calls `run_cli("init", db_path=db_path)` — this creates the SQLite database.
3. Additional `run_cli("add", ...)` calls populate nodes with justifications.
4. The test calls the subcommand under test (e.g., `run_cli("retract", "a", db_path=db_path)`).
5. `run_cli` patches `sys.argv`, `sys.stdout`, `sys.stderr`, calls `main()`, and captures output.
6. `main()` parses args, opens the database, executes the command, prints results, and calls `sys.exit()`.
7. `run_cli` catches `SystemExit`, returns `(stdout, stderr, exit_code)`.
8. The test asserts on the return tuple.

For tests involving file I/O (imports, exports, source hashing), `tmp_path` provides scratch files and `monkeypatch.chdir` sets the working directory.

## Invariants

- **Every subcommand returns exit code 0 on success and 1 on error.** No other codes are tested.
- **Error messages go to stderr; success output goes to stdout.** Tests consistently check `err` for `"Error"` on failures and `out` for result strings on success.
- **`init` must precede all other commands.** Without it, the database doesn't exist. Most tests start with `run_cli("init", ...)` — the exception is `TestCmdUpdate`, which relies on `add` to implicitly initialize.
- **Retraction cascades propagate immediately.** When a premise is retracted, its dependents go OUT in the same operation, and the cascade is reported in stdout.
- **Access tags enforce visibility.** Commands with `--visible-to` that don't match a node's tags must return exit code 1 with `"Access denied"`.
- **Idempotent operations report their no-op status.** Retracting an already-OUT node says `"already OUT"`; asserting an already-IN node says `"already IN"`.
- **`what-if` is read-only.** Output includes `"NOT modified"` to confirm the database wasn't changed.

## Error Handling

Errors are surfaced through two mechanisms:

1. **Exit code 1 + stderr message.** Missing nodes, access denied, file not found, duplicate IDs — all produce `code == 1` with a descriptive message in stderr containing `"Error"` or a specific string like `"Access denied"`, `"not found"`, `"--force"`.

2. **Graceful degradation for empty states.** Empty databases produce `"No nodes"`, `"No matching"`, `"No repos"`, `"No namespaces"`, `"No propagation events"` — all with exit code 0. These are not errors.

The test for `TestPropagateWithChanges` (line ~540) is notable: it directly manipulates the database via `Storage` to create an inconsistent state (node marked OUT when its justification says IN), then verifies that `propagate` fixes it. This tests the self-healing propagation path.

## Topics to Explore

- [file] `reasons_lib/cli.py` — The production code under test; contains `main()`, argument parsing, and all subcommand handlers
- [file] `reasons_lib/storage.py` — The SQLite persistence layer that every CLI command uses to load/save the belief network
- [file] `reasons_lib/network.py` — The TMS network data structure with propagation, retraction cascades, and nogood detection
- [function] `reasons_lib/cli.py:main` — The entrypoint that wires argparse subcommands to handler functions
- [general] `outlist-propagation-semantics` — How `--unless` (outlist) nodes interact with truth propagation — the GATE belief pattern tested in `TestExplainBranches`

## Beliefs

- `cli-tests-use-blackbox-main-patching` — Every CLI test invokes `main()` via `run_cli` with patched `sys.argv`/stdout/stderr rather than calling individual handler functions directly
- `cli-exit-code-zero-success-one-error` — All CLI subcommands use exit code 0 for success and exit code 1 for every error condition; no other exit codes appear in the test suite
- `cli-tests-are-self-contained` — Each test method initializes its own database via `run_cli("init")` and populates its own nodes; no shared state between tests
- `cli-errors-go-to-stderr` — Error messages (missing nodes, access denied, duplicate IDs) are written to stderr while success output goes to stdout
- `what-if-is-readonly` — The `what-if` subcommand simulates retract/assert without modifying the database, confirmed by asserting `"NOT modified"` in output

