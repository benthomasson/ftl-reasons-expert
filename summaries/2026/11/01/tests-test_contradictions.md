---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_contradictions.py

# `tests/test_contradictions.py`

## Purpose

This file is the test suite for the **contradictions detection subsystem** in `ftl-reasons` — a belief-network tool that uses LLM calls (via a Claude CLI subprocess) to find semantic conflicts (called "nogoods") between stored beliefs. The tests verify the full pipeline: formatting beliefs into prompts, parsing LLM responses into structured nogood objects, batch-processing logic, the public API surface, CLI integration, plan-file I/O, and an optional semantic-clustering code path.

The file owns the contract: *given a set of beliefs, the contradictions module must correctly identify pairs/groups that conflict, produce well-formed nogood records, and support a review-then-apply workflow via a markdown plan file.*

## Key Components

### Helper: `run_cli(*args, db_path=None)`
Patches `sys.argv`, `sys.stdout`, `sys.stderr` and calls `reasons.cli.main()`. Returns `(stdout, stderr, exit_code)`. This is the standard CLI-invocation harness used throughout the project's test suite.

### Helper: `_make_nodes()`
Returns a fixed 4-node dict: three `IN` beliefs (`premise-a`, `premise-b`, `premise-c`) and one `OUT` node (`out-node`). Serves as the canonical fixture for unit-level tests.

### `TestFormatBeliefsForContradictionCheck`
Tests `format_beliefs_for_contradiction_check(belief_ids, nodes)`:
- Renders each belief as `- \`id\`: text`
- Truncates text at 200 chars, appending `"..."`
- Silently skips IDs not present in `nodes`

### `TestParseContradictionResponse`
Tests `parse_contradiction_response(llm_output, valid_ids=None)`:
- Parses markdown sections starting with `### NOGOOD <id>`
- Extracts `claims` (comma-separated), `analysis`, `severity`
- **Rejects** sections with fewer than 2 claims
- **Filters** claims against `valid_ids` if provided; drops the entire nogood if fewer than 2 remain
- Returns `[]` for unparseable / empty responses

### `TestDetectContradictions`
Tests `detect_contradictions(nodes, belief_ids=None, batch_size=CONTRADICTION_BATCH_SIZE, timeout=None)`:
- Splits IN-beliefs into batches and calls the LLM subprocess once per batch
- Skips the LLM entirely when no IN beliefs exist
- Passes `timeout` through to `subprocess.run`
- Catches `RuntimeError` per-batch (one bad batch doesn't kill the rest)
- Excludes `OUT` nodes from the prompt
- When `belief_ids` is given, filters to only those IDs that are also IN

### `TestDetectContradictionsApi`
Tests `api.detect_contradictions(db_path, sample=None, auto_apply=False)`:
- Reads beliefs from the SQLite DB, filters to IN, optionally samples
- Returns a dict with keys `checked`, `found`, and (when auto_apply) `applied`
- `auto_apply=True` calls `api.add_nogood` for each detected conflict

### `TestCmdContradictions`
End-to-end CLI tests via `run_cli("contradictions", ...)`:
- Verifies the "No contradictions detected" message
- Verifies discovered nogoods are printed with `[NOGOOD] id (Severity)` format
- Verifies `-o plan.md` writes a structured plan file containing `# Contradiction Plan` header and `[APPLY]` markers

### `TestDetectContradictionsSemantic` *(conditional)*
Skipped unless `HAS_CLUSTER_DEPS` is true (requires `sentence-transformers` + `scikit-learn`). Tests `detect_contradictions_semantic(nodes)`:
- Groups semantically similar beliefs into clusters, then checks each cluster
- A single belief (no possible pair) short-circuits without calling the LLM
- Returns parsed nogoods the same way as the batch path

### `TestContradictionPlan`
Tests the plan-file round-trip:
- `api.write_contradiction_plan(contradictions, path, append=False)` — writes/overwrites a markdown plan
- `api.write_contradiction_plan(..., append=True)` — appends without repeating the header
- `api.parse_contradiction_plan(text)` — reads `[APPLY]` entries, ignores `[SKIP]` entries
- `api.apply_contradiction_plan(plan, db_path)` — inserts nogoods into the DB
- CLI `--accept plan.md` flow: applies only `[APPLY]` entries, reports count
- Edge cases: missing file → exit 1; empty plan → exit 0 with "No APPLY entries"

## Patterns

| Pattern | Where |
|---|---|
| **Mock-at-the-subprocess boundary** | Every LLM call is mocked by patching `reasons.llm.subprocess.run` and `reasons.llm.shutil.which`. No real Claude invocation ever happens. |
| **Fake result objects** | `type("R", (), {...})()` creates lightweight objects with `returncode`, `stdout`, `stderr` — avoiding a full `subprocess.CompletedProcess` dependency. |
| **Fixture-scoped DB** | `@pytest.fixture def db_path(self, tmp_path)` creates a throwaway SQLite file per test class, ensuring isolation. |
| **Conditional skip** | `@skip_no_cluster` gates the semantic tests behind optional deps, so CI without ML packages still passes. |
| **Layered test granularity** | Unit (format/parse) → Integration (detect + API) → E2E (CLI). Each layer mocks at a different boundary. |
| **Table-driven assertions** | E.g. `test_multiple_nogoods` asserts on a 2-element list by index rather than looping. |

## Dependencies

**This file imports:**
- `reasons.contradictions` — the module under test (`format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `detect_contradictions`, `CONTRADICTION_BATCH_SIZE`, `detect_contradictions_semantic`)
- `reasons.api` — public API for DB operations (`add_node`, `retract_node`, `detect_contradictions`, `write_contradiction_plan`, `parse_contradiction_plan`, `apply_contradiction_plan`, `init_db`)
- `reasons.cli` — the CLI entry point (`main`)
- `reasons.cluster` — optional; only `HAS_CLUSTER_DEPS` flag is read
- `reasons.llm` — indirectly, via the patch targets (`reasons.llm.subprocess.run`, `reasons.llm.shutil.which`)

**Nothing imports this file** — it is a leaf test module.

## Flow

```
CLI / API entry
       │
       ▼
 detect_contradictions(nodes, belief_ids?, batch_size?)
       │
       ├─ filter to IN-only beliefs
       ├─ chunk into batches of batch_size
       │
       ▼  (per batch)
 format_beliefs_for_contradiction_check → prompt string
       │
       ▼
 reasons.llm → subprocess.run(claude CLI, input=prompt)
       │
       ▼
 parse_contradiction_response(stdout, valid_ids)
       │  └─ split on "### NOGOOD", extract fields
       │  └─ filter claims to valid_ids
       │  └─ drop if < 2 claims remain
       │
       ▼
 aggregated list of nogood dicts
       │
       ├─► API: return {checked, found, applied?}
       ├─► CLI: print to stdout, optionally write plan file
       └─► auto_apply: api.add_nogood(...) for each
```

For the **semantic** path, `detect_contradictions_semantic` first clusters beliefs (via sentence-transformers), then feeds each cluster through the same format→LLM→parse pipeline, but with the guarantee that clustered beliefs are semantically related (reducing false negatives across the full graph).

## Invariants

1. **Minimum 2 claims per nogood.** `parse_contradiction_response` enforces this both before and after `valid_ids` filtering. A single-claim "contradiction" is structurally invalid.
2. **Only `IN` beliefs are checked.** `OUT` nodes are excluded from both the prompt and the valid-ID set. The tests `test_filters_to_in_only` and `test_empty_network_returns_empty` pin this.
3. **Batch independence.** One batch raising `RuntimeError` must not prevent other batches from completing (`test_batch_failure_continues`).
4. **Plan file idempotency on append.** Writing with `append=True` must not duplicate the `# Contradiction Plan` header (`test_write_contradiction_plan_append`).
5. **`[APPLY]` is the only actionable marker.** `parse_contradiction_plan` ignores `[SKIP]` and any unmarked entries.
6. **Text truncation at 200 chars.** `format_beliefs_for_contradiction_check` caps each belief's text to 200 characters + `"..."` to keep LLM prompts bounded.

## Error Handling

| Scenario | Behavior |
|---|---|
| LLM subprocess raises `RuntimeError` | Caught per-batch inside `detect_contradictions`; remaining batches still execute; results from failed batches are simply absent. Test: `test_batch_failure_continues`. |
| LLM returns unparseable text (no `### NOGOOD` sections) | `parse_contradiction_response` returns `[]`; the CLI prints "No contradictions detected." Test: `test_malformed_response_returns_empty`. |
| LLM references non-existent belief IDs | Filtered out by `valid_ids`; if fewer than 2 survive, the nogood is dropped. Test: `test_filters_nonexistent_claim_ids`. |
| `--accept` with missing plan file | CLI exits with code `1`. Test: `test_accept_missing_file`. |
| `--accept` with empty plan file | CLI exits `0`, prints "No APPLY entries". Test: `test_accept_empty_plan`. |
| Missing optional deps (`sentence-transformers`) | `HAS_CLUSTER_DEPS` is `False`; the entire `TestDetectContradictionsSemantic` class is skipped. No exception propagates. |
| `timeout` exceeded on subprocess | Propagates as a subprocess timeout (not explicitly tested here, but the `timeout` parameter is verified to be passed through in `test_timeout_passed_through`). |

Notably, **no test asserts on stderr content** for failure cases — errors are signalled via exit codes and the absence of expected stdout, not by parsing error messages.

## Topics to Explore

- [file] `reasons/contradictions.py` — The module under test; understand the actual prompt template, batch-chunking logic, and how `detect_contradictions_semantic` clusters beliefs before calling the LLM.
- [function] `reasons/api.py:apply_contradiction_plan` — The write-path that turns a parsed plan into actual `nogood` rows in the SQLite DB; understand what "applying" a nogood means for downstream belief retraction.
- [file] `reasons/cluster.py` — Provides `HAS_CLUSTER_DEPS` and the semantic clustering that the `TestDetectContradictionsSemantic` tests exercise; understand how sentence embeddings group beliefs.
- [file] `tests/conftest.py` — Shared fixtures and any global mocks (e.g., DB setup, LLM stubs) that these tests rely on implicitly.
- [general] `nogood-lifecycle` — Trace the full lifecycle: detection → plan file → human review (APPLY/SKIP) → apply → downstream effects on belief validity and dependent inferences.

## Beliefs

- `parse-contradiction-response-requires-min-2-claims` — A NOGOOD section with fewer than 2 valid claim IDs is silently dropped; it is never returned, even if the LLM explicitly named only one claim.
- `detect-contradictions-excludes-out-nodes` — Beliefs with `truth_value == "OUT"` are never included in the LLM prompt nor in the `valid_ids` filter set; they are invisible to contradiction detection.
- `plan-file-append-preserves-single-header` — Calling `write_contradiction_plan` with `append=True` must not emit a second `# Contradiction Plan` heading; the invariant is `text.count("# Contradiction Plan") == 1` after multiple appends.
- `batch-failure-is-non-fatal` — A `RuntimeError` from one LLM batch in `detect_contradictions` is caught and swallowed; subsequent batches still execute and their results are included in the returned list.
- `semantic-path-skips-llm-for-single-belief` — `detect_contradictions_semantic` returns `[]` without invoking the subprocess when only one IN belief exists, because a contradiction requires at least two claims.
