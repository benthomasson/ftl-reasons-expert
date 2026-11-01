---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_repair.py

# `tests/test_repair.py` — Test Suite for Smuggled-Premise Repair

## 1. Purpose

This file is the test suite for `reasons/repair.py`, which implements a **smuggled-premise repair workflow**. The core problem it addresses: a derived belief (e.g., "the water boiled") may be flagged as invalid during review because it implicitly relies on a premise (e.g., "water boils at 100 °C") that was never listed in its antecedents. The repair module's job is to:

1. Ask an LLM to **extract** the smuggled claim from the review context.
2. **Search** the belief network for premises that might express that claim.
3. Ask the LLM to **match** the extracted claim against the candidates.
4. **Link** the matched premise as a new antecedent on the justification.

This test file exercises every layer of that pipeline — the response parsers, the LLM-orchestration functions, the top-level `repair_smuggled_beliefs` coordinator, and the CLI entry point.

## 2. Key Components

### Helper Utilities

| Symbol | Role |
|--------|------|
| `run_cli(*args, db_path=None)` | Sandboxed CLI executor. Patches `sys.argv`, `stdout`, `stderr`, calls `reasons.cli.main()`, captures the `SystemExit` code. Returns `(stdout, stderr, exit_code)`. |
| `_make_repair_nodes()` | Factory that returns a dict of 4 nodes: three raw premises (`premise-a/b/c`) and one derived belief (`derived-boil`) whose SL justification cites `premise-b` and `premise-c` as antecedents. This is the canonical "smuggled premise" fixture — the derived belief assumes "water boils at 100 °C" but never lists `premise-a` as an antecedent. |

### Test Classes (ordered by abstraction level)

| Class | What it tests |
|-------|---------------|
| `TestParseExtractResponse` | `parse_extract_response()` — strips whitespace, quotes, and "Smuggled claim:" prefixes from raw LLM text. |
| `TestParseMatchResponse` | `parse_match_response()` — parses JSON from LLM prose, validates `matched_ids` against a candidate set, tolerates malformed JSON. |
| `TestExtractSmuggledClaim` | `extract_smuggled_claim()` — verifies the LLM subprocess is invoked with the right context and the returned stdout is parsed. |
| `TestFindMatchingPremises` | `find_matching_premises()` — same LLM-call pattern, plus the early-return path when the candidate list is empty. |
| `TestRepairSmuggledBeliefs` | `repair_smuggled_beliefs()` — the **orchestration** function. 10 test methods covering the full happy path, dry-run, every failure status, filtering rules, outlist preservation, and batch processing. |
| `TestCmdRepairSmuggled` | The `repair-smuggled` CLI subcommand. Uses a real temp DB (`tmp_path`), writes a review JSON file, and asserts on printed output and exit codes. |

### Contract of `repair_smuggled_beliefs` (inferred from tests)

```python
def repair_smuggled_beliefs(
    review_results: list[dict],   # [{"id", "valid", "comment"}, ...]
    nodes: dict[str, dict],       # full belief network
    db_path: str,
    dry_run: bool = False,
    search_fn: Callable[[str], str] = ...,  # returns JSON string of candidates
) -> list[dict]
```

Each element of the returned list has at minimum:
- `"status"`: one of `"repaired"`, `"no_candidates"`, `"no_match"`, `"extraction_failed"`, `"error"`
- `"matched_premises"`: list of premise IDs (only when status is `"repaired"`)
- `"error"`: message (only when status is `"error"`)

## 3. Patterns

- **Subprocess mocking via `side_effect` lists.** Because the LLM is invoked through `subprocess.run` (patched at `reasons.llm.subprocess.run`), sequential calls are simulated by passing a list to `side_effect`. Order matters: e.g., in `test_full_repair_flow`, the first call is the extraction LLM, the second is the matching LLM.

- **Ad-hoc result objects.** Instead of `MagicMock`, the tests construct lightweight fake result objects:
  ```python
  type("R", (), {"returncode": 0, "stdout": "...", "stderr": ""})()
  ```
  This is a deliberate choice — it makes the contract explicit (only `returncode`, `stdout`, `stderr` are read) and avoids the "anything goes" nature of `MagicMock`.

- **`patch` targets are implementation-internal.** The tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, meaning the LLM invocation lives inside `reasons/llm.py` and `repair.py` delegates to it. This couples the tests to that internal path.

- **Status-code enum via strings.** Rather than exceptions, the repair function communicates failure through a `"status"` field. Tests assert on the string, never on raised exceptions.

- **CLI tests as integration tests.** `TestCmdRepairSmuggled` uses `tmp_path` to create a real SQLite DB, writes a JSON review file, and drives the full CLI path — in contrast to the unit tests above it which fully mock the DB layer.

## 4. Dependencies

**Imports (this file → other modules):**
- `reasons.repair` — the module under test (5 public functions)
- `reasons.api` — `add_justification` is patched; also used in CLI tests to seed a real DB
- `reasons.cli` — `main()` entry point for CLI tests
- `reasons.llm` — indirectly, via patch targets (`shutil.which`, `subprocess.run`)

**Imported by (other modules → this file):**
- None directly. As a test file, it is only invoked by the test runner (`pytest`). It is listed in `tests/__init__.py`'s package but has no runtime dependents.

## 5. Flow

The primary orchestration flow tested in `TestRepairSmuggledBeliefs::test_full_repair_flow`:

```
review_results (list of {id, valid: False, comment})
        │
        ▼
  For each invalid belief:
        │
        ├─► Look up node in `nodes` dict
        │     └─ not found? → status="error"
        │
        ├─► extract_smuggled_claim(context, comment)
        │     └─ LLM call #1 (subprocess.run)
        │     └─ parse_extract_response(stdout)
        │     └─ empty? → status="extraction_failed"
        │
        ├─► search_fn(query) → JSON candidate list
        │     └─ filter: remove existing antecedents
        │     └─ filter: remove derived beliefs (have justifications)
        │     └─ empty after filter? → status="no_candidates"
        │
        ├─► find_matching_premises(claim, candidates)
        │     └─ LLM call #2 (subprocess.run)
        │     └─ parse_match_response(stdout, candidate_ids)
        │     └─ matched_ids empty? → status="no_match"
        │
        └─► api.add_justification(..., sl=matched_ids, unless=outlist)
              └─ dry_run? → skip, status="repaired"
              └─ else → apply, status="repaired"
        │
        ▼
  Return list of per-belief result dicts
```

The CLI flow (`TestCmdRepairSmuggled::test_from_review_file`) adds a front-end: read `--review-file` JSON → load the network from the DB → call `repair_smuggled_beliefs` → print a summary line per belief (e.g., `"REPAIRED derived-ab ← premise-b"`).

## 6. Invariants

These are the rules the tests enforce:

1. **Only `valid=False` beliefs are processed.** `test_skips_valid_beliefs` passes a `valid=True` entry and expects zero repairs.
2. **Existing antecedents are excluded from candidates.** `test_skips_existing_antecedents` — if `premise-b` is already in the justification's antecedent list and the search returns `premise-b`, it is filtered out, yielding `"no_candidates"`.
3. **Derived beliefs (nodes that have their own `justifications`) are excluded from candidates.** `test_skips_derived_beliefs_in_candidates` — a node with non-empty `justifications` is treated as a conclusion, not a premise.
4. **The original outlist is preserved.** `test_preserves_outlist` — when a justification has `outlist: ["blocker-x"]`, the repaired justification must pass `unless="blocker-x"` to `add_justification`.
5. **`matched_ids` are validated against the candidate set.** `TestParseMatchResponse::test_filters_invalid_ids` — any ID not in the candidate set is silently dropped.
6. **Malformed LLM JSON degrades gracefully.** `test_malformed_json` and `test_no_matched_ids_key` — the parser returns `{"matched_ids": []}` rather than raising.
7. **CLI requires a review file or explicit belief IDs.** `test_no_args_error` — without `--review-file` and no belief IDs, exit code is 1.

## 7. Error Handling

The code uses a **status-string protocol** rather than exceptions for domain-level failures:

| Condition | Status | Where tested |
|-----------|--------|--------------|
| Belief ID not in network | `"error"` + `"error"` message | `test_belief_not_in_network` |
| LLM extraction returns blank | `"extraction_failed"` | `test_extraction_failed` |
| Search returns no candidates (or all filtered) | `"no_candidates"` | `test_no_candidates`, `test_skips_existing_antecedents` |
| LLM matching returns no valid IDs | `"no_match"` | `test_no_match` |
| Dry-run | `"repaired"` but `add_justification` not called | `test_dry_run_no_apply` |

**Parser-level errors** (malformed JSON, missing keys) are swallowed inside `parse_match_response` and return a default empty structure. No exception propagates to the caller.

**CLI-level errors** are communicated via non-zero exit codes (`SystemExit(1)`) and a message on stderr/stdout.

**LLM subprocess errors** are not explicitly tested here — the mock always returns `returncode: 0`. This is a gap: there is no test for a non-zero `returncode` or a `subprocess.CalledProcessError`.

---

## Topics to Explore

- [file] `reasons/repair.py` — The module under test. Read the actual implementation of `repair_smuggled_beliefs`, `extract_smuggled_claim`, and `find_matching_premises` to see prompt construction, the search integration, and how the status codes are assigned.
- [file] `reasons/llm.py` — The LLM invocation layer. The tests patch `reasons.llm.subprocess.run` and `reasons.llm.shutil.which`, so understanding how the CLI tool is resolved and the subprocess is launched (timeout handling, environment, model selection) is critical for debugging real failures.
- [file] `reasons/repair_premises.py` — A sibling module (`test_repair_premises.py` also exists). Compare its API and repair strategy to `repair.py` to understand whether there are two distinct repair paths or a refactor in progress.
- [file] `reasons/api.py` — Contains `add_justification`, which is the mutation point. Understanding its contract (does it validate that `sl` IDs exist? does it rebuild the outlist?) explains why the tests assert on `call_args` rather than re-reading the DB.
- [general] `llm-response-parsing-robustness` — Both `parse_extract_response` and `parse_match_response` are thin parsers around untrusted LLM output. Explore whether there are retry/fallback strategies elsewhere in the codebase, and whether the current "silently return empty" behavior is intentional or a source of silent repair failures.

## Beliefs

- `repair-status-is-never-exception` — `repair_smuggled_beliefs` communicates every domain-level failure (extraction failure, no candidates, no match, belief not found) via a `"status"` string in the result dict; it never raises for those conditions.
- `outlist-preserved-on-repair` — When a new antecedent is linked via `add_justification`, the existing `outlist` of the justification is forwarded as the `unless` parameter; it is never dropped or reset.
- `derived-beliefs-excluded-from-candidates` — A node that has a non-empty `justifications` list is treated as a conclusion and is filtered out of the candidate set before LLM matching; only "raw" premises (empty justifications) are eligible.
- `llm-mocked-at-subprocess-boundary` — All LLM-dependent tests patch `reasons.llm.subprocess.run` (and `shutil.which`), meaning the repair module's LLM calls are fully bypassed; no HTTP, no API key, and no network I/O occurs during testing.
- `cli-requires-review-file-or-belief-ids` — The `repair-smuggled` CLI subcommand exits with code 1 and a "provide" message if neither `--review-file` nor explicit belief IDs are supplied; it does not default to scanning the entire network.
