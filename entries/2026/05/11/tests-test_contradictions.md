# File: tests/test_contradictions.py

**Date:** 2026-05-11
**Time:** 13:00

## Purpose

This file is the test suite for the **LLM-powered contradiction detection** subsystem of `ftl-reasons`. It validates the full pipeline: formatting beliefs into prompts, parsing structured LLM responses back into nogood records, orchestrating batched LLM calls, and the plan-based review workflow that lets a human approve or reject detected contradictions before they're recorded in the TMS.

It does **not** test the LLM itself — every test mocks `subprocess.run` to return canned responses — so it's purely testing the orchestration, parsing, filtering, and CLI integration layers.

## Key Components

### `run_cli(*args, db_path=None)`

A test harness that invokes the CLI entrypoint (`reasons_lib.cli:main`) in-process by patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns `(stdout, stderr, exit_code)`. Used by both `TestCmdContradictions` and `TestContradictionPlan` to test the CLI surface without spawning subprocesses.

### `_make_nodes()`

Factory that builds a minimal belief network: three IN premises (`premise-a`, `premise-b`, `premise-c`) and one OUT node (`out-node`). This is the canonical fixture for unit-level tests that operate on raw node dicts rather than a database.

### Test Classes

| Class | What it covers |
|---|---|
| `TestFormatBeliefsForContradictionCheck` | Prompt formatting — list rendering, 200-char truncation, graceful skip of missing node IDs |
| `TestParseContradictionResponse` | Response parsing — extracting NOGOOD sections, enforcing ≥2 claims, filtering against a valid-ID set, handling malformed output |
| `TestDetectContradictions` | Orchestration — batching, IN-only filtering, explicit ID filtering, timeout passthrough, batch-failure resilience |
| `TestDetectContradictionsApi` | API-level integration — `api.detect_contradictions()` with a real SQLite DB, sample-size limiting, auto-apply flow |
| `TestCmdContradictions` | CLI subcommand — output formatting, plan file generation |
| `TestDetectContradictionsSemantic` | Semantic clustering variant — embedding-based grouping, single-belief short-circuit (conditionally skipped if `sentence-transformers` / `scikit-learn` aren't installed) |
| `TestContradictionPlan` | Plan file round-trip — write/parse/apply cycle, append mode, `[APPLY]`/`[SKIP]` filtering, CLI `--accept` flow |

## Patterns

**Mock-at-the-boundary**: All LLM interaction is mocked at `reasons_lib.llm.subprocess.run` and `reasons_lib.llm.shutil.which`. The tests never touch a real model, which keeps them fast and deterministic.

**Inline mock objects**: Instead of `MagicMock`, the tests construct lightweight result objects via `type("R", (), {...})()` — an anonymous class with the three attributes `subprocess.run` returns (`returncode`, `stdout`, `stderr`). This is more explicit than a mock and makes the expected contract visible.

**Layered testing**: The same feature is tested at three levels — pure functions (`format_beliefs_for_contradiction_check`, `parse_contradiction_response`), the orchestrator (`detect_contradictions`), and the API/CLI surface (`api.detect_contradictions`, `run_cli("contradictions", ...)`). Each layer adds real infrastructure (SQLite DB, CLI arg parsing) while keeping the LLM mocked.

**Conditional skip**: `TestDetectContradictionsSemantic` is guarded by `@skip_no_cluster`, which checks whether `sentence-transformers` and `scikit-learn` are installed. This lets the test suite run in lean environments.

**tmp_path fixtures**: All tests that need a database use pytest's `tmp_path` to get an isolated directory, avoiding cross-test pollution.

## Dependencies

**Imports from the project:**
- `reasons_lib.contradictions` — the module under test (`format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `detect_contradictions`, `detect_contradictions_semantic`, `CONTRADICTION_BATCH_SIZE`)
- `reasons_lib.api` — higher-level API for DB operations (`add_node`, `retract_node`, `detect_contradictions`, `write_contradiction_plan`, `parse_contradiction_plan`, `apply_contradiction_plan`, `init_db`)
- `reasons_lib.cli` — CLI entrypoint (`main`)
- `reasons_lib.cluster` — optional ML dependency flag (`HAS_CLUSTER_DEPS`)

**Transitive mock targets:**
- `reasons_lib.llm.subprocess.run` — the actual LLM invocation
- `reasons_lib.llm.shutil.which` — Claude binary discovery

**Nothing imports this file** — it's a leaf test module.

## Flow

1. **Formatting**: `format_beliefs_for_contradiction_check` takes a list of belief IDs and a node dict, produces a markdown-formatted string listing each belief. Text is truncated at 200 chars with `...` suffix.

2. **LLM call**: `detect_contradictions` filters to IN-only beliefs, splits them into batches of `CONTRADICTION_BATCH_SIZE`, and sends each batch to the LLM. Each batch is an independent call that can fail without aborting the others.

3. **Parsing**: `parse_contradiction_response` extracts `### NOGOOD <id>` sections from the LLM's response, pulling out claims, analysis, and severity. It enforces ≥2 valid claims per nogood.

4. **Plan workflow**: Results are written to a markdown plan file (`write_contradiction_plan`) with `[APPLY]` tags. The user edits the file — changing `[APPLY]` to `[SKIP]` to reject false positives — then runs `reasons contradictions --accept plan.md` to apply the surviving nogoods to the database.

5. **Semantic variant**: `detect_contradictions_semantic` uses embedding-based clustering to group related beliefs before sending them to the LLM, so the LLM gets semantically coherent batches rather than arbitrary slices.

## Invariants

- **Only IN beliefs are checked**: OUT nodes are always filtered out before LLM calls, regardless of whether they appear in an explicit `belief_ids` list.
- **Minimum two claims**: A NOGOOD with fewer than two valid claims is silently dropped — both during initial parsing and after filtering against `valid_ids`.
- **Batch isolation**: A failed batch (exception from `subprocess.run`) doesn't abort the remaining batches — it's caught and skipped, returning an empty list for that batch.
- **Plan round-trip fidelity**: `write_contradiction_plan` → `parse_contradiction_plan` preserves IDs and claims for `[APPLY]` entries and drops `[SKIP]` entries.
- **Append preserves header**: When `append=True`, `write_contradiction_plan` adds new NOGOODs without duplicating the `# Contradiction Plan` header.
- **Text truncation at 200 chars**: `format_beliefs_for_contradiction_check` hard-truncates belief text at 200 characters with a `...` suffix.

## Error Handling

- **LLM failures are swallowed**: `test_batch_failure_continues` confirms that `RuntimeError` from `subprocess.run` results in an empty list, not a crash. This is a deliberate design choice — contradiction detection is advisory, not critical path.
- **Missing nodes are skipped silently**: `format_beliefs_for_contradiction_check` ignores IDs not found in the node dict.
- **Malformed LLM output returns empty**: If the LLM doesn't produce any `### NOGOOD` sections, `parse_contradiction_response` returns `[]`.
- **CLI exit codes**: `--accept` with a missing file returns exit code 1; an empty plan returns exit code 0 with an informational message.

## Topics to Explore

- [file] `reasons_lib/contradictions.py` — The implementation being tested: prompt construction, batching logic, and the semantic clustering variant
- [function] `reasons_lib/api.py:apply_contradiction_plan` — How accepted nogoods are recorded into the TMS database and what side effects they trigger
- [file] `reasons_lib/cluster.py` — The embedding/clustering layer that powers `detect_contradictions_semantic`, and when `HAS_CLUSTER_DEPS` is false
- [function] `reasons_lib/llm.py:subprocess.run` — The actual LLM invocation path that all these tests mock; understanding timeout handling and binary discovery
- [general] `nogood-semantics-in-tms` — How nogoods function in Doyle's TMS: what happens when a nogood is recorded, how it interacts with justification-based retraction

## Beliefs

- `contradiction-detection-filters-out-nodes` — `detect_contradictions` and `detect_contradictions_semantic` both exclude OUT-truth-value nodes before sending any prompt to the LLM, even if those IDs are explicitly requested
- `parse-contradiction-requires-two-valid-claims` — `parse_contradiction_response` drops any NOGOOD entry that has fewer than two claims after filtering against `valid_ids`
- `batch-llm-failure-returns-empty-not-raises` — When a batch's `subprocess.run` call throws an exception, `detect_contradictions` catches it and continues with remaining batches, returning an empty result for the failed batch
- `contradiction-plan-round-trips-apply-entries` — Writing a contradiction plan and parsing it back preserves all `[APPLY]`-tagged NOGOOD entries with their IDs and claims, while discarding `[SKIP]`-tagged entries
- `belief-text-truncated-at-200-chars` — `format_beliefs_for_contradiction_check` truncates any belief text longer than 200 characters, appending `...` as a suffix

