---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_research.py

# `tests/test_research.py`

## 1. Purpose

This file is the test suite for the **research/repair pipeline** — the system's mechanism for remedying beliefs that a reviewer flagged as invalid. The pipeline has a three-step decision workflow:

1. **Triage** — an LLM classifies the problem into one of four patterns: `search_and_link`, `soften`, `abandon`, or `research`.
2. **Act** — each pattern triggers a distinct repair action (link a missing premise, rewrite the claim weaker, retract the node, or mark it for manual investigation).
3. **Record** — metadata is stamped on the node so the repair is auditable.

The tests cover the full stack: raw JSON parsers → single-belief LLM calls → multi-belief orchestration (`repair_beliefs`) → the public API wrapper (`api.research`) → the CLI command (`reasons research`).

## 2. Key Components

### Helpers

| Helper | Contract |
|--------|----------|
| `run_cli(*args, db_path=None)` | Invokes `reasons.cli.main()` with a patched `sys.argv`/`stdout`/`stderr`. Returns `(stdout_str, stderr_str, exit_code)`. Catches `SystemExit` to capture the code. |
| `_make_nodes()` | Returns a fixed 4-node belief graph: three premises (`premise-a/b/c`) and one derived node (`derived-boil`) justified by an SL rule over `premise-b` and `premise-c`. All have `truth_value: "IN"`. |
| `_mock_result(stdout)` | Factory for a lightweight object mimicking `subprocess.run`'s return: `.returncode`, `.stdout`, `.stderr`. |
| `_mock_search(results)` | Returns a zero-arg callable that serialises `results` to JSON — used to stub the `search_fn` parameter. |

### Test Classes

- **`TestParseTriageResponse`** — Verifies `parse_triage_response` extracts `pattern` and `rationale` from JSON (including JSON embedded in prose), and returns `{"pattern": ""}` for invalid patterns or malformed input.

- **`TestParseSoftenResponse`** — Same shape for `parse_soften_response`; expects `softened_text` and `rationale`.

- **`TestComputeDepth`** — Tests `_compute_depth(node_id, nodes)`: premises → 0, one-level derived → 1, two-level chain → 2, missing node → 0.

- **`TestTriageBelief`** — Mocks the LLM subprocess and asserts `triage_belief` returns the parsed pattern. Also asserts depth and flagged-ancestor counts appear in the prompt sent to the LLM.

- **`TestSoftenBelief`** — Asserts `soften_belief` returns the LLM's `softened_text`.

- **`TestResearchBeliefs`** (the largest class, ~12 tests) — Exercises `repair_beliefs` end-to-end for every pattern plus failure modes:
  - `test_search_and_link` — 3 sequential LLM calls (triage → extract → match), asserts `add_justification` is called and metadata is stamped.
  - `test_soften` — 2 LLM calls, asserts `supersede_with_text` is called and the new node gets `"softened"` metadata.
  - `test_abandon` — 1 LLM call, asserts `retract_node` is called with `"abandoned"` in the reason.
  - `test_research_pattern` — 1 LLM call, asserts two metadata calls (`repair_research` + `repair_action`).
  - `test_dry_run_no_mutations` — All mutation APIs must be no-ops.
  - `test_triage_failed` / `test_soften_failed` / `test_search_and_link_extraction_failed` — Degraded LLM outputs produce distinct status strings.
  - `test_multiple_beliefs_different_patterns` — Two beliefs in one batch get independent triage paths.
  - `test_belief_not_found` — Non-existent ID → `"error"` status.
  - `test_skips_valid_beliefs` — `valid: True` entries are filtered out before any LLM call.

- **`TestApiResearch`** — Integration-level tests using `tmp_path` and a real SQLite DB via `api.add_node`. Verifies the `belief_ids` and `review_file` input paths, and the count invariant `total_invalid == linked + softened + abandoned + failed + errors`.

- **`TestCmdResearch`** — CLI tests: `--help`, missing-args error (exit 1), `--review-file` happy path, `--dry-run` (verifies node is unchanged in DB after run).

- **`TestBackwardCompatAlias`** — Asserts `research_beliefs is repair_beliefs` and `api.research is api.repair`. These are rename aliases kept for backward compatibility.

- **`TestCmdRepair`** — Mirrors `TestCmdResearch` for the `reasons repair` subcommand (the older name).

## 3. Patterns

- **Subprocess mocking via `side_effect` lists**: Because `reasons.llm` shells out to a CLI (e.g., `claude`) via `subprocess.run`, tests patch `reasons.llm.subprocess.run` with a list of `_mock_result` objects consumed in order. This models multi-turn LLM conversations (triage → extract → match) without hitting a real model.

- **`shutil.which` patching**: Every LLM test also patches `reasons.llm.shutil.which` to return `"/usr/bin/claude"`, satisfying the guard that "the binary must exist before we attempt to run it."

- **Table-driven per-pattern tests**: Each triage pattern gets its own test method in `TestResearchBeliefs`, making it easy to add a 5th pattern later.

- **Dry-run as a cross-cutting concern**: Multiple tests explicitly verify that `dry_run=True` suppresses all side-effect API calls (`retract_node`, `add_justification`, `set_metadata`) while still returning the computed result.

- **Integration via `tmp_path`**: `TestApiResearch` and `TestCmdResearch` use pytest's `tmp_path` fixture to create a real SQLite database, exercising the full `api.add_node → api.research` path. This catches schema/wiring bugs that unit mocks would hide.

- **Alias/compat tests**: `TestBackwardCompatAlias` is a guard against accidental removal of the `research_beliefs` / `api.research` shims during a refactor.

## 4. Dependencies

**This file imports from:**

| Module | Symbols used |
|--------|-------------|
| `reasons.repair` | `parse_triage_response`, `parse_soften_response`, `triage_belief`, `soften_belief`, `repair_beliefs`, `research_beliefs`, `_compute_depth` |
| `reasons.api` | `add_node`, `add_justification`, `set_metadata`, `supersede_with_text`, `retract_node`, `show_node`, `research` (alias), `repair` (alias) |
| `reasons.cli` | `main` |
| `reasons.llm` | (indirectly — patched at `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`) |
| `unittest.mock`, `io.StringIO`, `json`, `sys` | Standard test tooling |

**What depends on this file:** Nothing at runtime — it's a leaf test module. It is discovered by pytest via the `tests/` package and `conftest.py`.

## 5. Flow

The central execution path under test is `repair_beliefs(review_results, nodes, ...)`:

```
for each belief in review_results:
  skip if valid == True
  ┌─ LLM call 1: triage_belief()
  │   → parse_triage_response() → pattern
  │
  ├─ pattern == "search_and_link":
  │   ├─ LLM call 2: extract missing premise text
  │   ├─ search_fn(query) → candidate results
  │   ├─ LLM call 3: match candidates → matched_ids
  │   └─ api.add_justification(derived_id, matched_id, ...)
  │
  ├─ pattern == "soften":
  │   ├─ LLM call 2: soften_belief() → softened_text
  │   └─ api.supersede_with_text(old_id, softened_text)
  │
  ├─ pattern == "abandon":
  │   └─ api.retract_node(id, reason="...abandoned...")
  │
  ├─ pattern == "research":
  │   └─ api.set_metadata(id, "repair_research", rationale)
  │       api.set_metadata(id, "repair_action", "research")
  │
  └─ parse failure / LLM error:
      → status = "triage_failed" | "soften_failed" | "extraction_failed"
```

The CLI (`reasons research --review-file ...`) loads the review JSON, loads the belief graph, calls `repair_beliefs`, prints a summary (SOFTENED / ABANDONED / LINKED counts), and exits 0 or 1.

## 6. Invariants

- **Valid beliefs are never processed.** `test_skips_valid_beliefs` confirms `valid: True` entries produce zero results and zero LLM calls.
- **Dry-run is side-effect-free.** `test_dry_run_no_mutations` and `test_research_dry_run_no_metadata` assert that no `retract_node`, `add_justification`, `supersede_with_text`, or `set_metadata` call is made when `dry_run=True`.
- **API count consistency.** `test_counts_add_up` enforces: `total_invalid == linked + softened + abandoned + failed + errors`.
- **Backward-compat aliases are identity, not copies.** `research_beliefs is repair_beliefs` and `api.research is api.repair` must hold (identity check, not equality).
- **Premise depth is always 0.** `_compute_depth` returns 0 for nodes with no SL justifications, regardless of other fields.

## 7. Error Handling

The code under test (and these tests) follows a **no-raise, status-string** convention:

| Failure mode | Result status | Where asserted |
|---|---|---|
| LLM returns non-JSON for triage | `"triage_failed"` | `test_triage_failed` |
| LLM returns empty `softened_text` | `"soften_failed"` | `test_soften_failed` |
| LLM returns whitespace for extraction | `"extraction_failed"` | `test_search_and_link_extraction_failed` |
| Belief ID not in graph | `"error"` + `"not found"` message | `test_belief_not_found` |
| Invalid triage pattern string | `parse_triage_response` returns `{"pattern": ""}` → falls through to `triage_failed` | `test_invalid_pattern_ignored` |
| CLI called with no required args | `SystemExit(1)` | `test_no_args_error` |

Notably, `parse_triage_response` and `parse_soften_response` **never raise** — they return a dict with empty-string defaults. This lets `repair_beliefs` treat any malformed LLM output uniformly as a "failed" status rather than an exception.

---

## Topics to Explore

- [file] `reasons/repair.py` — The production module under test; contains the triage/soften/abandon orchestration logic, prompt construction, and the `repair_beliefs` loop that these tests exercise.
- [function] `reasons/repair.py:repair_beliefs` — The central orchestrator that iterates over invalid beliefs, dispatches to the correct pattern handler, and collects result records. Understanding its branching logic is prerequisite to reading these tests.
- [file] `reasons/llm.py` — Wraps the subprocess call to the LLM CLI. Tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`; knowing the real contract (prompt format, timeout, model selection) makes the mocks easier to interpret.
- [file] `reasons/api.py` — Provides `supersede_with_text`, `retract_node`, `add_justification`, `set_metadata` — the mutation surface that `repair_beliefs` calls. The `research`/`repair` aliases live here.
- [general] `belief-graph-depth` — How `_compute_depth` is used in triage prompts (deeper beliefs are more likely to be abandoned) and whether there's a maximum-depth guard elsewhere in the codebase.

---

## Beliefs

- `repair-beliefs-skips-valid` — `repair_beliefs` filters out any review result with `valid == True` before issuing an LLM call; valid beliefs produce no entry in the results list.
- `dry-run-prevents-mutations` — When `dry_run=True`, `repair_beliefs` computes and returns the intended action but does not call `retract_node`, `add_justification`, `supersede_with_text`, or `set_metadata`.
- `api-count-invariant` — The `api.research` result dict always satisfies `total_invalid == linked + softened + abandoned + failed + errors`.
- `research-beliefs-is-repair-beliefs` — `reasons.repair.research_beliefs` is an identity alias for `repair_beliefs` (same object, not a wrapper), preserved for backward compatibility with callers using the older name.
- `triage-parse-never-raises` — `parse_triage_response` and `parse_soften_response` return a dict with empty-string defaults for missing/malformed fields; they never raise, so callers rely on checking the `pattern` value rather than catching exceptions.
