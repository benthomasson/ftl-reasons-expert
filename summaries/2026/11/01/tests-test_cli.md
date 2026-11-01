---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_cli.py

# `tests/test_cli.py` — CLI Integration Test Suite

## 1. Purpose

This file is the **end-to-end integration test suite** for the `reasons` CLI. It exercises every user-facing command by invoking `reasons.cli.main()` exactly as a terminal user would, asserting on stdout/stderr content and the process exit code. It is the primary safety net that guarantees the CLI contract: correct subcommand parsing, correct output formatting, correct exit codes, and correct database state transitions.

It does **not** test internal logic in isolation (those live in `test_storage.py`, `test_network.py`, etc.). Its scope is: *given a sequence of CLI invocations against a fresh SQLite DB, does the system behave as the user expects?*

## 2. Key Components

### `run_cli(*args, db_path=None)` — The Test Harness

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

**Contract:**
- Returns `(stdout: str, stderr: str, exit_code: int)`
- `exit_code` is `0` if `main()` returns normally, or `e.code` if `main()` calls `sys.exit(code)`
- All I/O is captured in-memory; no real terminal is touched
- `db_path` is injected via `--db` flag (positional-arg style)
- The program name is always `"reasons"` (index 0 of argv)

### `db_path` Fixture

```python
@pytest.fixture
def db_path(tmp_path):
    return str(tmp_path / "test.db")
```

Provides a unique, ephemeral SQLite path per test. Every test class receives it, ensuring full isolation — no shared mutable state between tests.

### Test Classes (grouped by CLI subcommand)

| Class | Commands Covered |
|-------|-----------------|
| `TestInit` | `init` |
| `TestAdd` | `add` |
| `TestAddJustification` | `add-justification` |
| `TestRetractAssert` | `retract`, `assert` |
| `TestStatus` | `status` |
| `TestShow` | `show` |
| `TestExplain` | `explain` |
| `TestTrace` / `TestTraceAccessTags` | `trace`, `trace-access-tags` |
| `TestList` | `list` |
| `TestSearchLookup` | `search`, `lookup` |
| `TestExport` | `export`, `export-markdown` |
| `TestCompact` | `compact` |
| `TestChallenge` | `challenge`, `defend` |
| `TestNogood` | `nogood` |
| `TestWhatIf` | `what-if` |
| `TestSupersede` | `supersede` |
| `TestCheckIntegrity` | `check-integrity` |
| `TestBackfillHashes` | `backfill-hashes` |
| `TestConvertToPremise` | `convert-to-premise` |
| `TestPropagate` / `TestPropagateWithChanges` | `propagate` |
| `TestLog` / `TestLogWithEntries` | `log` |
| `TestRepos` | `add-repo`, `repos` |
| `TestImportExportJson` | `export`, `import-json` |
| `TestSummarize` | `summarize` |
| `TestImportBeliefs` / edge cases | `import-beliefs` |
| `TestImportAgent` / edge cases | `import-agent`, `sync-agent` |
| `TestNamespaces` | `namespaces` |
| `TestDeduplicate` / `TestDeduplicateSemantic` | `deduplicate` |
| `TestCheckStale` / `TestHashSources` | `check-stale`, `hash-sources` |
| `TestDerive` | `derive` |
| `TestAccept` / `TestAcceptEdgeCases` | `accept` |
| `TestProposalLifecycle` | `propose-*`, `accept-proposal`, `reject-proposal`, `withdraw-proposal`, `proposals`, `proposal`, `propose-update` |
| `TestNamesOnlyFormat` | `--format names-only` across multiple commands |
| `TestNoCommand` | bare invocation, `--version` |
| `TestErrorCases` / `TestTraceEdgeCases` / others | Error paths across commands |

## 3. Patterns

### a) Arrange-Act-Assert via Sequential `run_cli` Calls

Every test follows the same rhythm:

```python
run_cli("init", db_path=db_path)          # arrange: fresh DB
run_cli("add", "a", "A", db_path=db_path) # arrange: seed data
out, err, code = run_cli("show", "a", ...) # act: the command under test
assert code == 0                           # assert: exit code
assert "ID:     a" in out                  # assert: output contract
```

There are no setup/teardown hooks beyond the fixture. State is built by *calling the CLI itself*, which means tests are also regression tests for the full write-path.

### b) Direct Storage Manipulation for Edge Cases

A few tests bypass the CLI to create states that are hard (or impossible) to reach through normal commands:

```python
# test_check_integrity — simulate tampering
store = Storage(db_path)
net = store.load()
net.nodes["p1"].text = "Tampered"
store.save(net)
store.close()

# test_propagate_with_changes — force inconsistent state
net.nodes["b"].truth_value = "OUT"
store.save(net)
```

This is the only place `reasons.storage.Storage` is imported directly. It's used to *corrupt* or *mutate* the DB behind the CLI's back to test integrity checks and repair logic.

### c) Mocking for LLM-Dependent Paths

```python
with patch("reasons.propose_update.invoke_model", return_value=mock_response):
    out, err, code = run_cli("propose-update", "a", ...)
```

The single LLM call site is mocked so tests remain deterministic and don't require API keys.

### d) Conditional Skipping for Optional Dependencies

```python
try:
    from reasons.cluster import HAS_CLUSTER_DEPS
except ImportError:
    HAS_CLUSTER_DEPS = False

@pytest.mark.skipif(not HAS_CLUSTER_DEPS, ...)
class TestDeduplicateSemantic:
```

Semantic deduplication requires `sentence-transformers` + `scikit-learn`. The entire class is skipped if those aren't installed, keeping the suite runnable in minimal environments.

### e) `--visible-to` Access Control Testing

Multiple test classes (`TestStatus`, `TestShow`, `TestExplain`, `TestTrace`, `TestTraceAccessTags`, `TestTraceEdgeCases`) verify that `--access-tags` on nodes + `--visible-to` on the query produce correct inclusion/exclusion. The pattern is always: add a node with `--access-tags finance`, then query with `--visible-to hr` and assert denial (exit 1 + "Access denied" in stderr).

### f) `--format names-only` as a Cross-Cutting Concern

`TestNamesOnlyFormat` at the bottom tests a single flag across five different commands (`status`, `search`, `list`, `compact`, `trace`, `explain`), asserting that the output is bare node IDs with no decorative text. This is a **contract test** for scripted/programmatic consumption of the CLI.

## 4. Dependencies

### Imports (what this file uses)

| Module | Role |
|--------|------|
| `reasons.cli.main` | The function under test |
| `reasons.storage.Storage` | Direct DB manipulation in 3–4 tests |
| `reasons.cluster.HAS_CLUSTER_DEPS` | Feature-detection for optional deps |
| `reasons.propose_update.invoke_model` | Patched in `test_propose_update_store` |
| `unittest.mock.patch` | All sys.* and LLM mocking |
| `pytest` (fixture, mark.skipif) | Test framework |
| `io.StringIO` | Captured output |
| `json`, `pathlib.Path`, `re` | Parsing/validating output, file I/O |

### What depends on this file

Nothing imports `tests/test_cli.py`. It is a **leaf consumer** — it sits at the bottom of the dependency graph and is executed by the test runner.

## 5. Flow

```
pytest collects test_cli.py
  └─ For each test method:
       1. Fixture `db_path` creates a unique temp SQLite path
       2. Test calls run_cli("init", ...) → main() → creates DB + schema
       3. Test calls run_cli("add", ...) one or more times → main() → mutates DB
       4. Test calls run_cli("<command-under-test>", ...) → main() → reads/mutates DB, writes to captured stdout
       5. Test asserts on (stdout, stderr, exit_code)
       6. (Optional) Test calls Storage directly to corrupt state, then re-runs a command
       7. tmp_path is cleaned up by pytest automatically
```

`main()` is called synchronously within the same process. `SystemExit` is the *only* exception it's expected to raise; any other exception propagates and fails the test (which is intentional — unexpected exceptions should crash the test).

## 6. Invariants

1. **Exit code semantics:** `0` = success, `1` = user error (bad args, missing node, access denied, duplicate ID). The test suite never expects exit code `2` or any other value, reinforcing that the CLI uses a binary success/failure model.

2. **Idempotency of `init`:** Running `init` twice without `--force` must fail with exit 1 and mention `--force` in stderr. This prevents accidental data loss.

3. **Cascade propagation:** Retracting a premise must cause its derived dependents to go OUT, and the output must name those dependents (`"b" in out`). Similarly, asserting a retracted node must cascade IN.

4. **Access-tag isolation:** A node with `--access-tags finance` is invisible to `--visible-to hr` queries. This is tested across `show`, `explain`, `trace`, `status`, and `trace-access-tags` — the invariant is that **no command leaks access-restricted content to unauthorized viewers**.

5. **`what-if` is read-only:** `test_what_if_retract` asserts `"NOT modified" in out`, confirming the command simulates without persisting.

6. **Proposal lifecycle atomicity:** A proposal must be either `pending` → `accepted` (state changes) or `pending` → `rejected`/`withdrawn` (state unchanged). `test_reject_proposal` explicitly verifies the node remains IN after rejection.

7. **`names-only` format purity:** When `--format names-only` is used, the output must contain *only* node IDs — no brackets, no status words, no labels. This is a strict contract for machine-parseable output.

## 7. Error Handling

The CLI's error model is **exit-code + stderr message**, not exceptions to the caller:

- **Missing node:** `run_cli("show", "missing")` → exit 1, `"Error" in err`
- **Access denied:** exit 1, `"Access denied" in err`
- **Duplicate add:** exit 1, `"Error" in err`
- **Invalid combination** (e.g., `supersede` with both positional `new_id` *and* `--text`): exit 1, `"cannot specify both" in err`
- **Missing required argument** (e.g., `propose-nogood` with < 2 nodes): exit 1, `"at least 2" in err`
- **File not found** (e.g., `import-beliefs /nonexistent/beliefs.md`): exit 1, `"Error" in err`

The test suite **never catches exceptions from `main()`** other than `SystemExit`. If `main()` raised a `KeyError` or `ValueError`, the test would crash with an unhandled exception — which is the desired behavior (it signals a bug, not a user error).

One subtle pattern: `test_accept_stale_drift` verifies that accepting a proposal whose target node has *already changed state* (drift) produces a `"STALE"` warning but still returns exit 0. This means stale proposals don't hard-fail; they warn.

---

## Topics to Explore

- [file] `reasons/cli.py` — The module under test; understanding its argument parsing (likely `argparse` or `click`) and how it dispatches to per-command handler functions will clarify why each test asserts specific stdout strings.
- [file] `reasons/storage.py` — `Storage.load()` / `Storage.save()` are used directly in 4 tests to corrupt state; understanding the schema and serialization format is essential for writing new integrity tests.
- [function] `reasons/propose_update.py:invoke_model` — The single LLM integration point; knowing its I/O contract (JSON array of proposals) is needed to write additional mocked tests for the `propose-update` command.
- [file] `reasons/network.py` — The in-memory belief graph; cascade logic (retract/assert propagation) and `truth_value` transitions live here, and the CLI tests in `TestRetractAssert` and `TestWhatIf` are its behavioral spec.
- [general] `access-tags-enforcement` — How `--access-tags` and `--visible-to` interact across all read commands; the tests in this file define the full matrix, but the implementation in `reasons/api.py` or `reasons/storage.py` reveals where the filtering actually occurs.

## Beliefs

- `cli-exit-code-1-signals-user-error` — Every user-facing failure (missing node, access denied, duplicate ID, bad arguments) produces exit code 1 and a human-readable message on stderr; exit code 0 is reserved for success.
- `run-cli-mocks-all-sys-io` — The `run_cli` helper patches `sys.argv`, `sys.stdout`, and `sys.stderr` simultaneously, meaning no test in this file can accidentally write to the real terminal or read the real command line.
- `what-if-does-not-mutate-state` — The `what-if` subcommand is contractually read-only; `test_what_if_retract` asserts the output contains "NOT modified", and no subsequent `show` in that test verifies a state change.
- `names-only-format-is-machine-parseable` — When `--format names-only` is passed, the output is a newline-delimited list of bare node IDs with zero decorative text, enabling direct use in shell scripts and agent pipelines.
- `access-tags-enforce-isolation-at-cli-layer` — A node tagged `finance` is invisible to any command invoked with `--visible-to hr`, regardless of whether the command is `show`, `explain`, `trace`, or `status`; denial always produces exit 1 + "Access denied".
