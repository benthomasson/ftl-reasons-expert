# File: tests/test_contradictions.py

**Date:** 2026-05-08
**Time:** 14:05

# `tests/test_contradictions.py`

## Purpose

This file tests the LLM-powered contradiction detection pipeline — the system that scans a belief network for logically inconsistent beliefs (nogoods). It validates three layers: the formatting/parsing functions that prepare data for and interpret responses from an LLM, the `detect_contradictions` orchestration function that batches and dispatches LLM calls, and the CLI `contradictions` subcommand that ties it all together with output formatting and file writing.

## Key Components

### `run_cli(*args, db_path=None)`

A test helper that invokes the CLI's `main()` in-process by patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns a `(stdout, stderr, exit_code)` tuple. This avoids subprocess overhead and lets tests inspect output directly. The `db_path` kwarg injects `--db <path>` so each test gets an isolated SQLite database.

### `_make_nodes()`

Factory that builds a minimal four-node belief network: three IN premises (`premise-a`, `premise-b`, `premise-c`) and one OUT node (`out-node`). This is the shared fixture for unit-level tests that operate on raw node dicts rather than a database.

### `TestFormatBeliefsForContradictionCheck`

Tests the function that renders a subset of beliefs into a markdown-formatted string for the LLM prompt. Validates three contracts:
- Produces `- \`id\`: text` lines for each requested belief
- Truncates belief text to 200 characters (with `...` suffix) to stay within prompt budgets
- Silently skips IDs not present in the nodes dict (no KeyError)

### `TestParseContradictionResponse`

Tests the parser that extracts structured nogood records from the LLM's markdown response. The expected format is `### NOGOOD <id>` headers followed by `- Claims:`, `- Analysis:`, `- Severity:` fields. Key invariants validated:
- Each nogood must have **at least two claims** — single-claim entries are dropped
- When `valid_ids` is supplied, claim IDs not in that set are filtered out, and if filtering reduces claims below two, the entire nogood is dropped
- Malformed responses (no `### NOGOOD` headers) return an empty list
- Multiple nogoods in one response are all extracted

### `TestDetectContradictions`

Tests the orchestration function that batches IN beliefs and sends each batch to the LLM. Key behaviors:
- **Batching**: 3 IN beliefs with `batch_size=2` produces exactly 2 LLM calls
- **IN-only filtering**: OUT nodes are excluded from the prompt even if present in the network
- **Specific ID filtering**: When `belief_ids` is provided, only those IDs (intersected with IN nodes) are checked
- **Timeout passthrough**: The `timeout` parameter reaches the subprocess call
- **Fault tolerance**: If the LLM subprocess raises `RuntimeError`, the batch is skipped and an empty list is returned (no crash)

### `TestDetectContradictionsApi`

Integration-level tests using a real SQLite database (via `tmp_path`). Validates the `api.detect_contradictions` wrapper:
- Respects IN/OUT status from the database
- `sample=N` limits how many beliefs are checked
- `auto_apply=True` calls `add_nogood` to record detected contradictions in the database

### `TestCmdContradictions`

End-to-end tests for the `reasons contradictions` CLI subcommand:
- Prints "No contradictions detected" when clean
- Formats found nogoods as `[NOGOOD] id (Severity)` with claims and analysis
- `--output` writes a markdown report file with proper headers
- `--dry-run` combined with `--auto-apply` prevents nogoods from being applied (applied count = 0)

## Patterns

**LLM isolation via mocking**: Every test patches two things: `reasons_lib.llm.shutil.which` (to pretend `claude` CLI exists) and `reasons_lib.llm.subprocess.run` (to inject canned LLM responses). This makes the test suite fully deterministic and offline.

**Anonymous result objects**: Mock subprocess results are created inline with `type("R", (), {...})()` — a quick way to build objects with `.returncode`, `.stdout`, `.stderr` attributes without defining a class or importing `SimpleNamespace`.

**Layered testing**: The file tests the same pipeline at three granularities — pure functions (format/parse), orchestration (detect_contradictions with node dicts), API (detect_contradictions with a database), and CLI (full subcommand). Each layer adds integration surface.

**Isolated databases via `tmp_path`**: The pytest `tmp_path` fixture provides a fresh directory per test, so `api.add_node` creates a clean SQLite database with no cross-test contamination.

## Dependencies

**Imports:**
- `reasons_lib.contradictions` — the module under test (`format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `detect_contradictions`, `CONTRADICTION_BATCH_SIZE`)
- `reasons_lib.api` — used in integration tests to create nodes, retract nodes, and call `api.detect_contradictions`
- `reasons_lib.cli.main` — the CLI entrypoint, invoked by `run_cli`
- `reasons_lib.llm` — not imported directly, but patched at `reasons_lib.llm.subprocess.run` and `reasons_lib.llm.shutil.which`

**Imported by:** Nothing — this is a test module.

## Flow

1. **Unit tests** call `format_beliefs_for_contradiction_check` or `parse_contradiction_response` directly with constructed inputs, asserting on string content or parsed dicts.
2. **Orchestration tests** call `detect_contradictions(nodes, ...)` with mocked LLM subprocess. The function filters to IN beliefs, batches them, formats each batch into a prompt, sends it via `subprocess.run`, and parses the response. Tests verify call counts, prompt contents, and parameter passthrough.
3. **API tests** create a real database with `api.add_node`, optionally retract nodes, then call `api.detect_contradictions` which loads the network from SQLite and delegates to `detect_contradictions`.
4. **CLI tests** use `run_cli("contradictions", ...)` which patches `sys.argv` and calls `main()`. The CLI parses args, calls the API layer, and formats output to stdout or a file.

## Invariants

- A nogood requires **at least 2 valid claims**. This is enforced by `parse_contradiction_response` and tested from multiple angles (single claim, all-fake claims, filtering reduces below two).
- Only **IN beliefs** are candidates for contradiction checking. OUT beliefs are excluded regardless of whether they appear in `belief_ids` or the full network.
- LLM failures in any batch **do not abort** the remaining batches — the system degrades gracefully to an empty result.
- `--dry-run` **overrides** `--auto-apply` — when both flags are set, nothing is written to the database.

## Error Handling

- **LLM subprocess failure**: `detect_contradictions` catches exceptions per-batch and continues, returning whatever results it gathered from successful batches (or an empty list if all fail). Tested in `test_batch_failure_continues`.
- **Missing node IDs**: `format_beliefs_for_contradiction_check` silently skips IDs not found in the nodes dict. `parse_contradiction_response` filters claims against `valid_ids` and drops nogoods that fall below the two-claim minimum.
- **Malformed LLM output**: If the LLM returns text without `### NOGOOD` headers, the parser returns an empty list — no crash, no false positives.
- **CLI exit codes**: `run_cli` captures `SystemExit` and returns the exit code, so tests can assert on both output and exit status.

## Topics to Explore

- [file] `reasons_lib/contradictions.py` — The production implementation of format, parse, and detect functions tested here
- [function] `reasons_lib/api.py:detect_contradictions` — The API-layer wrapper that loads the network from SQLite and delegates to the core function
- [file] `reasons_lib/llm.py` — The LLM subprocess invocation layer that these tests mock; understanding its interface explains the mock shape
- [general] `nogood-lifecycle` — How detected nogoods flow from LLM output through `add_nogood` into the belief network and trigger retraction cascades
- [function] `reasons_lib/cli.py:main` — The CLI dispatch logic that wires the `contradictions` subcommand to the API

## Beliefs

- `contradiction-min-two-claims` — `parse_contradiction_response` drops any nogood with fewer than 2 valid claim IDs; this is enforced both before and after `valid_ids` filtering
- `contradiction-in-only-filter` — `detect_contradictions` excludes OUT nodes from LLM prompts even when explicitly listed in `belief_ids`
- `contradiction-batch-fault-tolerance` — If the LLM subprocess raises an exception for one batch, `detect_contradictions` continues processing remaining batches and returns partial results
- `contradiction-dry-run-overrides-auto-apply` — When both `--dry-run` and `--auto-apply` are passed to the CLI, no nogoods are recorded in the database (applied count is 0)

