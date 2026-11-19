---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_contradictions.py

# `tests/test_contradictions.py`

## Purpose

This is the test suite for the **LLM-powered contradiction (nogood) detection** feature of the `reasons` package. It exercises three layers of the stack:

1. **The `reasons.contradictions` module** — prompt formatting, LLM-response parsing, batching, and semantic clustering (`format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `detect_contradictions`, `detect_contradictions_semantic`).
2. **The `reasons.api` façade** — `api.detect_contradictions`, `api.write_contradiction_plan`, `api.parse_contradiction_plan`, `api.apply_contradiction_plan` (network-aware wrappers around the module).
3. **The CLI surface** — the `reasons contradictions` subcommand, including the `-o` plan-file output and the `--accept` apply flow.

The file is therefore a mixed unit / integration test suite: pure-function tests at the top, LLM-mocked orchestration tests in the middle, and end-to-end CLI tests at the bottom.

---

## Key Components

### Helpers

- **`run_cli(*args, db_path=None)`** — Wraps `reasons.cli.main()` in `sys.argv`/stdout/stderr patches and catches `SystemExit`, returning `(stdout, stderr, exit_code)`. This is the standard pattern for invoking the CLI in-process without spawning a subprocess.
- **`_make_nodes()`** — Returns a fixture dict of four nodes: three `IN` beliefs (`premise-a`, `premise-b`, `premise-c`) and one `OUT` node (`out-node`). Used to verify `IN`-only filtering.

### Unit-level test classes

- **`TestFormatBeliefsForContradictionCheck`**
  - Confirms output format: `` `- `id`: text`` (markdown bullet with backticked id).
  - Confirms truncation: a 300-char text yields a **200-char** string ending in `...` (so effectively `text[:197] + "..."`).
  - Confirms missing node ids are silently skipped.

- **`TestParseContradictionResponse`** — Exercises `parse_contradiction_response(response, valid_ids=None)`:
  - Parses `### NOGOOD <id>` blocks followed by `- Claims:` (comma-separated), `- Analysis:`, `- Severity:` lines.
  - **Requires ≥2 claims** after optional `valid_ids` filtering — otherwise the nogood is dropped.
  - `valid_ids` acts as an allow-list; unknown claim IDs are stripped.
  - Malformed / free-form responses (e.g. `"No contradictions detected."`) return `[]`.
  - Multiple `### NOGOOD` sections are parsed into a list, preserving order.
  - Supports 3+ claims (no upper bound).

### Orchestration tests

- **`TestDetectContradictions`** — Tests `detect_contradictions(nodes, belief_ids=None, batch_size=..., timeout=...)`:
  - **Batching**: 3 `IN` beliefs with `batch_size=2` → `subprocess.run` called twice.
  - **`IN`-only filtering**: `OUT` nodes never appear in the prompt.
  - **`belief_ids` narrowing**: caller-supplied ids are intersected with `IN` truth-values (an `OUT` id gets dropped, so only 1 batch fires).
  - **Timeout pass-through**: `timeout=600` is forwarded to `subprocess.run(..., timeout=600)`.
  - **Failure isolation**: a `RuntimeError` from `subprocess.run` is swallowed; the function returns `[]` rather than raising (important invariant — a flaky LLM must not crash the CLI).

- **`TestDetectContradictionsApi`** — Tests `api.detect_contradictions(db_path, sample=..., auto_apply=...)` against a real SQLite DB:
  - Reads `IN` nodes from the DB, drops retracted nodes.
  - `sample=N` caps how many beliefs are inspected (`result["checked"] == N`).
  - `auto_apply=True` calls `add_nogood` per parsed contradiction and reports `result["applied"] >= 1`.
  - Result dict contract: `{"checked": int, "found": int, "applied": int, ...}`.

- **`TestCmdContradictions`** — CLI end-to-end:
  - Prints `"No contradictions detected"` when the LLM returns no findings.
  - Prints `"[NOGOOD] <id> (<severity>)"`, claim list, and analysis per finding.
  - `-o <path>` writes a markdown plan file (see `TestContradictionPlan` for format) and prints an `--accept` hint on stdout.
  - `--accept <file>` applies `[APPLY]`-tagged entries.

- **`TestDetectContradictionsSemantic`** (skipped unless `HAS_CLUSTER_DEPS`) — Tests `detect_contradictions_semantic(nodes)`:
  - Groups beliefs by embedding cluster and issues one prompt per cluster.
  - Confirms related beliefs (`auth-login-validates` and `auth-login-skips-validation`) land in the same prompt.
  - Returns parsed nogoods from the LLM.
  - Short-circuits: empty network → `[]`; **single belief → `[]` with `subprocess.run` not called at all** (avoids wasting an LLM call).

### Plan file tests

- **`TestContradictionPlan`**
  - `api.write_contradiction_plan(contradictions, path, append=False)` produces a `# Contradiction Plan` document with `### NOGOOD <id> [APPLY]` sections and `- Claims:`, `- Analysis:`, `- Severity:` lines.
  - `append=True` preserves the header and appends further sections (the header appears exactly once).
  - `api.parse_contradiction_plan(text)` extracts only `[APPLY]` entries (not `[SKIP]`), returning `[{"id": ..., "claims": [...]}, ...]`.
  - `api.apply_contradiction_plan(plan, db_path)` returns `{"applied": N, "nogoods": [...]}`.
  - CLI `--accept` flow: existing plan + valid `[APPLY]` entries → `Applied 1 nogood`, exit 0; missing file → exit 1; empty plan → `"No APPLY entries"` message, exit 0.

---

## Patterns

- **LLM mocking at the subprocess boundary.** Every LLM call is stubbed by patching `reasons.llm.shutil.which` (so the "claude" binary detection passes) and `reasons.llm.subprocess.run` (so no real process spawns). The mock object mimics `subprocess.CompletedProcess` (`returncode`, `stdout`, `stderr`) via `type("R", (), {...})()`.
- **Prompt inspection via `mock_run.call_args[1]["input"]`.** Confirms `subprocess.run` is called with the prompt passed as `input=` (keyword), not positional.
- **Fixture-based DB setup.** `db_path` fixtures call `api.add_node(..., db_path=db)` and reuse the temp SQLite file across a test method.
- **In-process CLI capture.** `run_cli` patches `sys.argv`, `sys.stdout`, `sys.stderr`, and catches `SystemExit`, matching the convention used throughout the rest of `tests/`.
- **Conditional skip via capability detection.** `HAS_CLUSTER_DEPS` gates the semantic tests so the core suite still runs when `sentence-transformers`/`scikit-learn` aren't installed.
- **Black-box assertions on prompts and outputs.** Rather than mocking `contradictions.format_beliefs_for_contradiction_check`, the orchestration tests inspect the string that would be handed to the LLM — catching format regressions end-to-end.
- **Contract-first result dicts.** `api.*` return plain dicts (`{"checked", "found", "applied", "nogoods"}`) — tests assert on keys, not objects, so the wrapper has wide latitude on internals.

---

## Dependencies

**Imported by this file:**
- `reasons.contradictions` — `format_beliefs_for_contradiction_check`, `parse_contradiction_response`, `detect_contradictions`, `detect_contradictions_semantic`, `CONTRADICTION_BATCH_SIZE`.
- `reasons.api` — DB and plan-management façade.
- `reasons.cli.main` — CLI entry point.
- `reasons.cluster.HAS_CLUSTER_DEPS` — optional dep flag used for skipping.
- `reasons.llm` — the (patched) module that shells out to `claude`/LLM backends.

**Implicitly pinned by the tests (in `reasons/contradictions.py`, `reasons/api.py`, `reasons/cli.py`):**
- `detect_contradictions` must call `subprocess.run(..., input=prompt, timeout=...)`.
- `parse_contradiction_response` must accept a `valid_ids` kwarg.
- `api.detect_contradictions` must accept `db_path`, `sample`, `auto_apply`.
- `api.write_contradiction_plan` must accept `append=`.
- CLI must accept `-o` and `--accept`.

If you refactor any of those signatures, this file is the first to break.

---

## Flow

1. **Unit tests** call `format_beliefs_for_contradiction_check(ids, nodes)` and `parse_contradiction_response(response, valid_ids=...)` directly — pure functions, no I/O.
2. **`TestDetectContradictions`** builds an in-memory `nodes` dict, patches the LLM layer, calls `detect_contradictions(...)`, then inspects `mock_run.call_count` and `mock_run.call_args_list` (prompt contents, timeout kwarg).
3. **`TestDetectContradictionsApi`** creates a temp DB via a fixture, patches the LLM, calls `api.detect_contradictions(db_path=..., sample=..., auto_apply=...)`, and asserts on the result dict.
4. **`TestCmdContradictions`** uses `run_cli(...)`:
   - `argv = ["reasons", "--db", db_path, "contradictions", ...]`
   - `main()` runs, calls `api.detect_contradictions`, prints findings, optionally writes plan via `-o`.
   - `SystemExit.code` is captured and asserted.
5. **`TestDetectContradictionsSemantic`** either skips or executes the same mocking pattern against `detect_contradictions_semantic`.
6. **`TestContradictionPlan`** exercises the write → parse → apply pipeline, plus the CLI `--accept` path.

Data transformations:
```
nodes dict  →  [format_beliefs_for_contradiction_check]  →  prompt string
            →  subprocess.run (mocked)                    →  raw response
            →  parse_contradiction_response(valid_ids)    →  [{id, claims, analysis, severity}]
            →  api.add_nogood / write_contradiction_plan  →  DB rows / markdown plan
            →  api.apply_contradiction_plan               →  {applied, nogoods}
```

---

## Invariants

These are enforceable rules the tests will catch a regression of:

- **`IN`-only input.** Only nodes with `truth_value == "IN"` are considered; `OUT` nodes are never formatted into prompts or fed to the LLM.
- **Minimum two claims per nogood.** Responses with fewer than 2 valid claim IDs are discarded.
- **`valid_ids` is an allow-list.** Claim IDs not in `valid_ids` are stripped before the "≥2 claims" check; a nogood whose remaining claims drop below 2 is discarded.
- **Text truncation ceiling.** A formatted belief line's text portion (after `: `) is exactly 200 chars including the trailing `...`.
- **Batching.** `detect_contradictions` chunks `IN` beliefs by `batch_size` (default `CONTRADICTION_BATCH_SIZE`), issuing one `subprocess.run` per chunk.
- **Timeout forwarding.** The `timeout=` argument is passed through verbatim to `subprocess.run`.
- **Single-belief short-circuit for semantic mode.** `detect_contradictions_semantic` returns `[]` and never invokes the LLM when fewer than 2 `IN` beliefs are present.
- **Non-raising batch failures.** A `subprocess.run` exception is caught inside `detect_contradictions`; the function returns `[]` instead of propagating.
- **Plan format is stable.** Header `# Contradiction Plan` appears exactly once; sections are `### NOGOOD <id> [APPLY]`; fields are `- Claims:`, `- Analysis:`, `- Severity:`.
- **`[SKIP]` entries are honored.** `parse_contradiction_plan` extracts only `[APPLY]`.
- **`--accept` requires the file to exist.** Missing plan → exit code 1; empty/no-`[APPLY]` plan → exit 0 with `"No APPLY entries"` message.

---

## Error Handling

- **LLM failures are swallowed.** `test_batch_failure_continues` and `test_batch_failure_continues` (via `side_effect=RuntimeError`) verify that `detect_contradictions` returns `[]` rather than raising — a contract that keeps the CLI robust to model outages.
- **CLI exit codes are explicit.** `run_cli` captures `SystemExit.code`; tests assert `1` for missing `--accept` file and `0` for empty plan. Any `main()` path that raises without `SystemExit` would surface as a test error rather than a clean exit code.
- **Parser is forgiving of malformed input.** `parse_contradiction_response("No contradictions detected.")` returns `[]` — no exception. This means the LLM is free to respond conversationally, but only `### NOGOOD` blocks yield actionable results.
- **Missing nodes are silently skipped.** `format_beliefs_for_contradiction_check(["nonexistent"], nodes)` does not raise.
- **Retraction is respected.** `test_filters_to_in_only` (API variant) retracts `belief-c` and asserts `result["checked"] == 2` — stale `OUT` rows are excluded from the inspection count.

---

## Topics to Explore

- [file] `reasons/contradictions.py` — The module under test: prompt construction, regex/parser for `### NOGOOD` blocks, batch loop, and semantic clustering wrapper.
- [function] `reasons/contradictions.py:detect_contradictions_semantic` — Clustering strategy that groups semantically similar beliefs before prompting; explains why the single-belief short-circuit exists.
- [function] `reasons/api.py:detect_contradictions` — The DB-aware façade that resolves `sample`, filters `IN`, and orchestrates `auto_apply`; the source of the `{checked, found, applied}` contract.
- [function] `reasons/cli.py:main` — CLI dispatcher; shows how `contradictions`, `-o`, and `--accept` are wired and what exit codes the tests pin.
- [file] `reasons/llm.py` — Shells out to `claude`; the target of every `subprocess.run` mock — worth understanding `timeout=`/`input=` semantics if you plan to change the invocation surface.

## Beliefs

- `detect-contradictions-only-considers-in-truth-values` — Both `detect_contradictions` and its semantic variant skip any node whose `truth_value` is not `"IN"`.
- `nogood-requires-at-least-two-valid-claim-ids` — `parse_contradiction_response` discards any `### NOGOOD` block with fewer than 2 claims after applying the optional `valid_ids` allow-list.
- `detect-contradictions-swallows-batch-errors` — A `subprocess.run` exception during batching is caught and yields `[]` rather than propagating to the caller.
- `format-truncation-caps-text-at-200-chars` — Truncated belief text including the trailing `...` totals exactly 200 characters.
- `accept-flow-requires-existing-plan-file` — `reasons contradictions --accept <missing>` exits with code 1; an existing but empty plan exits 0 with a `"No APPLY entries"` message.
- `semantic-detection-skips-single-belief` — `detect_contradictions_semantic` returns `[]` and never calls `subprocess.run` when fewer than 2 `IN` beliefs exist.
