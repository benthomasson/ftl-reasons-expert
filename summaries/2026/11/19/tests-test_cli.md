---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_cli.py

# tests/test_cli.py

## Purpose

`tests/test_cli.py` is a pytest integration test suite for the `reasons` CLI. It does not unit-test individual command-handler functions directly; instead it drives the CLI entry point `reasons.cli.main()` with patched `sys.argv`, captures stdout/stderr, and asserts exit codes plus user-facing output.

Its job is to serve as a regression suite for the CLI surface and the end-to-end behavior behind it:

- command parsing and dispatch through `main()`
- SQLite persistence through real temporary databases
- CRUD and state transitions (`add`, `retract`, `assert`, `supersede`, etc.)
- access-control behavior via `--visible-to`
- import/export and markdown handling
- proposal lifecycle workflows
- integrity, stale-source, deduplication, propagation, and maintenance commands
- `--format names-only` output contracts

The suite targets the `reasons` package. Even though `reasons_lib/` mirrors many modules in the repository tree, this file imports `from reasons.cli import main`, so it exercises `reasons/`.

## Key Components

### `db_path(tmp_path)`

Module-level pytest fixture:

```python
@pytest.fixture
def db_path(tmp_path):
    return str(tmp_path / "test.db")
```

Contract:

- Returns a string path for a SQLite database under pytest’s per-test `tmp_path`.
- It does **not** create the database itself.
- Most tests explicitly call `run_cli("init", db_path=db_path)` before using it.
- `TestProposalLifecycle` overrides this fixture and pre-populates the DB.

### `run_cli(*args, db_path=None)`

This is the central test harness.

```python
def run_cli(*args, db_path=None):
    argv = ["reasons"]
    if db_path:
        argv += ["--db", db_path]
    argv += list(args)
    stdout, stderr = StringIO(), StringIO()
    with patch.object(sys, "argv", argv), \
         patch.object(sys, "stdout", stdout), \
         patch.object(sys, "stderr", stderr):
        try:
            main()
        except SystemExit as e:
            return stdout.getvalue(), stderr.getvalue(), e.code
    return stdout.getvalue(), stderr.getvalue(), 0
```

Contract:

- Simulates CLI invocation as `reasons [--db <db_path>] <args...>`.
- Inserts `--db` before the subcommand, matching the CLI’s expected global option ordering.
- Patches `sys.argv`, `sys.stdout`, and `sys.stderr` with `StringIO`.
- Catches `SystemExit` and returns its `.code`.
- If `main()` returns normally, returns exit code `0`.
- Returns `(stdout, stderr, exit_code)`.

Important consequence: tests assert numeric exit codes via `code == 0` or `code == 1`, not via `pytest.raises(SystemExit)`. Any non-`SystemExit` exception raised by a command will not be swallowed by `run_cli()` and will fail the test.

### `HAS_CLUSTER_DEPS`

At module load:

```python
try:
    from reasons.cluster import HAS_CLUSTER_DEPS
except ImportError:
    HAS_CLUSTER_DEPS = False
```

Used by:

```python
@pytest.mark.skipif(not HAS_CLUSTER_DEPS,
                    reason="sentence-transformers and scikit-learn not installed")
class TestDeduplicateSemantic:
```

This skips semantic deduplication tests when optional clustering dependencies are unavailable. Non-semantic deduplication tests still run.

### Test class groups

The file is organized into many focused pytest classes, usually one per command or feature area. Representative groups:

- Basic lifecycle: `TestInit`, `TestAdd`, `TestAddJustification`, `TestRetractAssert`
- Read/inspect: `TestStatus`, `TestShow`, `TestExplain`, `TestTrace`, `TestTraceAccessTags`, `TestList`, `TestSearchLookup`
- Import/export: `TestExport`, `TestImportExportJson`, `TestImportBeliefs`, `TestImportAgent`, `TestImportJsonWithNogoods`
- Maintenance/integrity: `TestCheckIntegrity`, `TestBackfillHashes`, `TestPropagate`, `TestLog`, `TestHashSources`, `TestCheckStale`
- State/proposal workflows: `TestSupersede`, `TestProposalLifecycle`, `TestAccept`, `TestAcceptEdgeCases`
- Deduplication: `TestDeduplicate`, `TestDeduplicateSemantic`, `TestDeduplicateAcceptPlan`
- Output formatting: `TestNamesOnlyFormat`
- Error/edge cases: `TestErrorCases`, `TestTraceEdgeCases`, `TestWhatIfEdgeCases`, `TestShowSource`, `TestConvertToPremiseEdgeCases`

### `TestProposalLifecycle.db_path`

This class redefines its own fixture:

```python
@pytest.fixture
def db_path(self, tmp_path):
    db = str(tmp_path / "test.db")
    run_cli("init", db_path=db)
    run_cli("add", "a", "Premise A", db_path=db)
    run_cli("add", "b", "Premise B", db_path=db)
    run_cli("add", "derived-ab", "AB combined", "--sl", "a,b", db_path=db)
    return db
```

Contract:

- Every test in `TestProposalLifecycle` starts with a fresh DB containing `a`, `b`, and `derived-ab`.
- This shadows the module-level `db_path` fixture only for that class.
- It reduces repeated arrange steps for proposal tests.

## Patterns

- **CLI-level integration testing**: tests exercise `main()` rather than importing and calling command handlers directly. This covers argparse wiring, command dispatch, stdout/stderr behavior, and exit codes.
- **Real temporary SQLite databases**: `tmp_path` gives isolation. Multi-step tests reuse the same DB path across multiple `run_cli()` calls to simulate a user session.
- **Fixture override pattern**: most tests manage `init` explicitly, while `TestProposalLifecycle` overrides `db_path` to preload data.
- **Black-box assertions with selective white-box setup**: most verification is through CLI output, but tests use direct `Storage` access to induce integrity failures, missing hashes, or inconsistent propagation state.
- **Mock external LLM dependency**: `propose-update` tests patch `reasons.propose_update.invoke_model` with a JSON response so no external model call is made.
- **Working-directory control**: source-file tests use `monkeypatch.chdir(tmp_path)` so relative `--source source.py` paths resolve inside the test temp directory.
- **Substring assertions rather than full-output snapshots**: tests generally assert key phrases like `"Added a [IN]"`, `"Went OUT"`, or `"Access denied"`. This is less brittle than exact output comparison, but it still makes user-facing wording part of the CLI contract.
- **Idempotent no-ops are success**: retracting an already-OUT node or asserting an already-IN node returns code `0` and prints `"already OUT"` / `"already IN"`.
- **Some tests are intentionally tolerant**: for example, `test_deduplicate_finds_similar` accepts either `"Cluster"` or `"No duplicate"`, and `test_retract_restoration_hints` only asserts restoration hint content if `"Note:"` is present. These tests are weaker than fully deterministic assertions.

## Dependencies

Top-level imports:

- `json`
- `sys`
- `io.StringIO`
- `unittest.mock.patch`
- `pytest`
- `reasons.cli.main`

Local imports inside tests:

- `pathlib.Path`
- `reasons.storage.Storage`
- `reasons.cluster.HAS_CLUSTER_DEPS`
- `re` and `json` in a few proposal tests

Production modules directly exercised or manipulated:

- `reasons.cli` through `main()`
- `reasons.storage.Storage` for DB tampering/backfill/propagation setup
- `reasons.cluster` for optional semantic dedupe detection
- `reasons.propose_update.invoke_model`, patched in `propose-update` tests

No production module imports `tests/test_cli.py`; pytest discovers and executes it. It depends on pytest and on the `reasons` package being importable.

## Flow

A typical test follows this flow:

1. pytest provides `tmp_path` and creates a fresh temporary directory.
2. The `db_path` fixture returns `str(tmp_path / "test.db")`.
3. The test calls `run_cli(...)` for setup, for example:
   - `run_cli("init", db_path=db_path)`
   - `run_cli("add", "a", "Premise A", db_path=db_path)`
4. `run_cli()` builds `["reasons", "--db", db_path, ...]`, patches `sys.argv`/stdout/stderr, and calls `main()`.
5. `main()` parses arguments and dispatches to a command handler. The handler opens the SQLite DB, mutates/loads the network, prints output, and either returns or calls `sys.exit(...)`.
6. If `SystemExit` is raised, `run_cli()` catches it and returns `(stdout, stderr, e.code)`.
7. The test asserts exit code and output substrings.
8. Repeated `run_cli()` calls in the same test share the same SQLite file, allowing stateful scenarios.

Special setup flows:

- **Integrity/backfill/propagate tests** directly open `Storage(db_path)`, load the network, mutate fields like `net.nodes["p1"].text`, `text_hash`, or `truth_value`, save, and close. Then they invoke the CLI command under test.
- **Proposal update tests** patch `reasons.propose_update.invoke_model` before invoking `propose-update`.
- **Stale-source tests** `chdir` into `tmp_path`, create/modify/delete `source.py`, and run `hash-sources` / `check-stale`.

## Invariants

- `init` creates a DB and prints `"Initialized"`.
- A second `init` without `--force` exits `1` and stderr contains `"--force"`.
- `add` rejects duplicate IDs with exit code `1` and stderr containing `"Error"`.
- `--db <path>` must be accepted as a global option before the subcommand, as generated by `run_cli`.
- Successful commands generally exit `0`; error paths generally exit `1`.
- Idempotent state transitions are treated as success:
  - retract already-OUT → `0`, `"already OUT"`
  - assert already-IN → `0`, `"already IN"`
  - what-if already-OUT/already-IN → `0`
- Access control is enforced on `status`, `show`, `explain`, `trace`, and `trace-access-tags` via `--visible-to`; denied reads exit `1` with `"Access denied"`.
- Retraction/assertion cascades are visible in output:
  - retracting a premise causes dependents to `"Went OUT"`
  - asserting it causes dependents to `"Went IN"`
  - adding a justification to an OUT node can print `"Cascade"`
- `what-if` is a dry run: it must report predicted changes and the test asserts `"NOT modified"`.
- `supersede` requires exactly one of `new_id` or `--text`; both or neither is an error.
- Proposal IDs are deterministic in tests, e.g. `prop-a-retract-1`, `prop-a-supersede-1`, `prop-new-c-add-1`. Proposal list counts like `"1 proposal(s)"` and `"2 proposal(s)"` are also asserted.
- Accepting a stale proposal after the target node has already changed returns code `0` and prints `"STALE"` rather than failing.
- `propose-nogood` requires at least two nodes and errors with `"at least 2"`.
- `export --output=-` must emit parseable JSON to stdout; `test_export_json` does `json.loads(out)`.
- `import-json` round-trips nodes and preserves nogoods in its output.
- `hash-sources` can backfill missing hashes and `--force` re-hashes them.
- `check-stale` detects changed and deleted sources, exits `1`, and prints `"STALE"` / `"DELETED"`.
- `check-integrity` detects unauthorized text mutation and prints the mutated node id.
- `--format names-only` produces one node name per line for `status`, `search`, `list`, `compact`, `trace`, and `explain`, without normal decorations such as `"Beliefs:"`, `"Status"`, or `[+]`.

## Error Handling

- `run_cli()` catches `SystemExit` and returns its code. This is how the tests inspect CLI failures without terminating pytest.
- CLI errors are generally written to `sys.stderr` and exit `1`. Tests assert snippets such as:
  - `"Error"`
  - `"Access denied"`
  - `"--force"`
  - `"cannot specify both"`
  - `"either new_id or --text is required"`
  - `"at least 2"`
- Some commands intentionally report success-like or warning behavior:
  - `accept` with skipped proposals exits `0` and may print `"SKIP"` to stderr or `"No valid proposals"` to stdout.
  - `accept-proposal` against a stale proposal exits `0` and prints `"STALE"`.
- Non-`SystemExit` exceptions are not caught by `run_cli()`; they fail the test immediately. This is intentional: uncaught exceptions are treated as CLI bugs.
- Optional dependency absence is handled by `HAS_CLUSTER_DEPS`; semantic dedupe tests skip rather than fail.
- Direct `Storage` setup failures or unexpected schema/attribute changes will fail tests before the CLI assertion, which makes those tests sensitive to storage-layer refactors.

## Topics to Explore

- [file] `reasons/cli.py` — Contains `main()` and the argparse dispatch that every test invokes indirectly.
- [function] `reasons/cli.py:main` — The entry point whose `sys.argv`, stdout/stderr, and `SystemExit` contract is assumed by `run_cli`.
- [file] `reasons/storage.py` — The persistence layer directly manipulated by integrity, backfill, and propagate tests.
- [file] `reasons/propose_update.py` — Hosts `invoke_model`, which is patched in `propose-update` tests to avoid real LLM calls.
- [general] `cli-access-control` — Investigate how `--visible-to`, access tags, and `Access denied` behavior are implemented across commands.

## Beliefs

- `run-cli-catches-systemexit` — `run_cli()` catches `SystemExit` and returns its `.code`, so tests assert numeric exit codes instead of expecting process termination.
- `cli-tests-use-real-temporary-sqlite-db` — Every DB-backed test uses a real SQLite file under pytest’s `tmp_path`, created either by `run_cli("init")` or by `TestProposalLifecycle`’s fixture.
- `global-db-option-precedes-subcommand` — `run_cli()` always inserts `--db <path>` immediately after `reasons` and before the command tokens.
- `export-dash-emits-json-on-stdout` — `export --output=-` is expected to print pure JSON to stdout, as verified by `json.loads(out)` in `test_export_json`.
- `proposal-lifecycle-fixture-preloads-three-nodes` — `TestProposalLifecycle.db_path` initializes a DB with `a`, `b`, and `derived-ab` before each test in that class.
- `semantic-dedupe-tests-skip-without-cluster-deps` — `TestDeduplicateSemantic` is skipped when `reasons.cluster.HAS_CLUSTER_DEPS` is false.
