---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_repair.py

# `tests/test_repair.py` — Explanation

## Purpose

This file is the acceptance/regression suite for the **smuggled-premise repair** feature implemented in `reasons/repair.py`. "Smuggling" is the failure mode where a derived belief's conclusion depends on a premise that was never listed as an antecedent. The repair workflow:

1. Take `review_results` output (beliefs flagged `valid=False` with a comment).
2. Ask an LLM to phrase the missing ("smuggled") premise as a discrete claim.
3. Search the network for candidate premises.
4. Filter candidates (drop existing antecedents, derived beliefs).
5. Ask an LLM to pick the matching ones.
6. Link them via `reasons.api.add_justification` — optionally only when `unless` (formerly `outlist`) blockers are not defeated.

The tests are valuable as **executable spec**: they pin down status strings, LLM-call ordering, dry-run semantics, candidate filtering, and CLI surface.

---

## Key Components

### Module-level helpers

- **`run_cli(*args, db_path=None)`** — A lightweight harness that patches `sys.argv`, `sys.stdout`, and `sys.stderr`, calls `reasons.cli.main()`, and converts `SystemExit` into a `(stdout, stderr, code)` triple. Note the argument order: `["reasons", "--db", <db>, *args]` — the global `--db` flag is inserted **before** the subcommand, which is required for argparse's global-vs-subparser resolution.
- **`_make_repair_nodes()`** — Builds a canonical four-node fixture:
  - `premise-a` "Water boils at 100C…" (the smuggled fact)
  - `premise-b`, `premise-c` (already in `derived-boil`'s `antecedents`)
  - `derived-boil` with a single `SL` justification using `[premise-b, premise-c]` and an empty `outlist`.
  This fixture is the setup for nearly every test in `TestRepairSmuggledBeliefs`.

### Test classes (each maps to one repair-layer function)

| Class | Target | What it locks down |
|---|---|---|
| `TestParseExtractResponse` | `parse_extract_response` | Plain text passthrough, quote/whitespace/`"Smuggled claim:"` prefix stripping, empty input → `""`, multiline input preserved (containment assertion). |
| `TestParseMatchResponse` | `parse_match_response` | JSON extraction from prose, filtering of IDs not in the allowed candidate set, empty `matched_ids`, malformed JSON → empty list, missing key → empty list. |
| `TestExtractSmuggledClaim` | `extract_smuggled_claim` | Delegates to `reasons.llm` (patched); prompt includes context; empty LLM output → `""`. |
| `TestFindMatchingPremises` | `find_matching_premises` | Calls the LLM; when candidates are empty it **short-circuits** (`subprocess.run` must not fire) and returns `{"matched_ids": [], "rationale": "...no candidates..."}`. |
| `TestRepairSmuggledBeliefs` | `repair_smuggled_beliefs` | The main orchestrator — see Flow below. |
| `TestCmdRepairSmuggled` | `reasons.cli:main` with `repair-smuggled` | `--help` exit 0, no-args error with "provide", end-to-end repair against a real temp SQLite DB, `--dry-run` behavior. |

### Mocking idioms used

- **Fake `subprocess.CompletedProcess`** via `type("R", (), {"returncode":0, "stdout":..., "stderr":""})()` — lighter than importing `subprocess`.
- **Sequential LLM responses** via `patch("reasons.llm.subprocess.run", side_effect=[extract_resp, match_resp, ...])`. The order of the `side_effect` list *is* the expected call order.
- **Injectable search** via the `search_fn` kwarg of `repair_smuggled_beliefs`; tests provide `lambda query, **kw: json.dumps(results)`. This decouples the orchestrator from a live search backend (network/pg/merkle).
- **`patch("reasons.api.add_justification")`** to assert the mutation is made, and to inspect `call_args[1]["sl"]` / `call_args[1]["unless"]`.

---

## Patterns

- **Dependency injection** — `search_fn` is a first-class parameter; the search backend is not imported at the top of `repair.py` but passed in. This is why unit tests can run with zero external I/O.
- **Layered mocking** — LLM transport (`subprocess.run`) and the domain operation (`api.add_justification`) are mocked *separately*, so each test can assert "was the LLM called?" and "was the DB written?" independently.
- **Status-string state machine** — every `repair_smuggled_beliefs` entry returns one of `repaired`, `no_candidates`, `no_match`, `extraction_failed`, `error`. Tests branch on that string rather than exception types, indicating the function is intentionally non-throwing on per-belief failures.
- **Fixture-as-scenario** — `_make_repair_nodes` returns a challenge setup that is mutated in tests (`test_preserves_outlist` adds `blocker-x`, `test_multiple_invalid_beliefs` adds `derived-steam`) rather than duplicating the base graph. Good: keeps the failure mode small and legible.
- **`call_count`-keyed mock search** — `test_multiple_invalid_beliefs` uses a closure state (`call_count = [0]`) to return different results for the first vs second belief. This is how the tests pin "the orchestrator processes beliefs in input order."

---

## Dependencies

**Imports (what it depends on):**
- `reasons.repair` — the module under test; exposes the five public symbols.
- `reasons` + `api` — used as `api.add_node(db_path=...)` and `api.add_justification(db_path=...)` in the "real integration" CLI tests, and patched in unit tests.
- `reasons.cli:main` — the CLI entry point invoked via `run_cli`.
- `reasons.llm.shutil.which` / `reasons.llm.subprocess.run` — the transport that is always patched. This file therefore **does not exercise real LLM calls**, which keeps CI hermetic.
- Standard library: `json`, `sys`, `io.StringIO`, `unittest.mock.patch`, `pytest.tmp_path`.

**Imported by (reverse deps):** Nothing product code imports a test file. It is only collected by `pytest` (discovery via `test_*.py` and `Test*` class convention).

---

## Flow

For the happy path, `repair_smuggled_beliefs(review_results, nodes, db_path, search_fn, dry_run)` executes **per invalid belief** in this order (verified by which `side_effect` entries are consumed and which are *not*):

1. **Filter** `review_results` to `valid=False` entries. `test_skips_valid_beliefs` asserts a `valid=True` result produces `len(repairs) == 0` — nothing is touched, no LLM call.
2. **Lookup** the belief in `nodes`. If missing → append `{"status": "error", "error": "...not found..."}` and stop processing this belief. `test_belief_not_in_network`.
3. **Extract**: `extract_smuggled_claim(context, comment)` → LLM call #1. If the parsed result is `""` → `extraction_failed`, return (search is *not* called). `test_extraction_failed` proves this by providing only one `side_effect` entry — a second LLM call would raise.
4. **Search**: `search_fn(claim, ...)` → JSON list of candidate dicts `{id, text, truth_value, source, match}`.
5. **Filter candidates**:
   - Drop any candidate whose `id` is already in the belief's `antecedents` (`test_skips_existing_antecedents` — `premise-b` is dropped).
   - Drop any candidate that is a **derived belief** (has non-empty `justifications`) — `test_skips_derived_beliefs_in_candidates`.
   - This filtering **must** happen before `find_matching_premises`; both negative tests provide only one `side_effect` (extract) and pass — proving the LLM match step never runs.
6. If the filtered set is empty → `no_candidates`. `test_no_candidates` uses empty search results.
7. **Match**: `find_matching_premises(claim, candidates)` → LLM call #2. Empty `matched_ids` → `no_match` (`test_no_match`).
8. **Apply** (unless `dry_run`): call `api.add_justification(<belief_id>, sl=["premise-a"], unless=<outlist[0]>, db_path=...)`. The `sl` kwarg is asserted in `test_full_repair_flow`; `unless` is asserted in `test_preserves_outlist` to equal the original `outlist` element `"blocker-x"`.
9. **Return** one result dict per processed invalid belief, in input order.

**Dry-run semantics:** `test_dry_run_no_apply` asserts the status is still `"repaired"` (the intent is reported) but `api.add_justification` was **never called**. So dry-run changes only the side effect, not the reported status.

**CLI flow** (`TestCmdRepairSmuggled`): `reasons repair-smuggled --review-file <path> [--dry-run] --db <path>`. The review file format is `{"results": [{"id", "valid", "comment"}]}` — see the shape in `test_from_review_file`. Output is text containing `"REPAIRED"` and the matched premise id. `--dry-run` output must contain `"dry run"` (lowercased comparison). Missing `--review-file`/`--belief` argument exits with code `1` and a message containing `"provide"`.

---

## Invariants

- **Only `valid=False` results are ever mutated.** `test_skips_valid_beliefs` → `len(repairs) == 0`.
- **Extraction short-circuits before search.** Empty LLM extract → `extraction_failed`; the second `subprocess.run` never fires (`side_effect` would raise).
- **Candidates are filtered before matching.** Antecedent and derived-belief candidates both collapse to `no_candidates` with only one `side_effect` provided.
- **Statuses are exhaustive and observable:** `repaired`, `no_candidates`, `no_match`, `extraction_failed`, `error`.
- **`add_justification` receives `sl` containing the matched premise ids** and `unless` derived from the original `SL` justification's `outlist`.
- **`dry_run=True` never mutates the store** even when status == `"repaired"`.
- **Belief ordering is deterministic** — results array preserves `review_results` input order (`test_multiple_invalid_beliefs` asserts `repairs[0]` is `derived-boil` and `repairs[1]` is `derived-steam`).
- **`parse_match_response` never trusts the LLM's ids.** Any id not in the allowed set is silently dropped (defensive against hallucinated ids).

---

## Error Handling

- **Per-belief errors are captured, not raised.** `belief_not_in_network` yields a result dict with `"status": "error"` and an `"error"` string containing `"not found"`. The orchestrator returns normally, so one bad review entry can't abort a batch.
- **Malformed LLM JSON is swallowed.** `parse_match_response("no json here", ...)` returns `{"matched_ids": []}` — the downstream maps that to `no_match`. Similarly a missing `matched_ids` key is treated as empty.
- **Empty/whitespace LLM extract is swallowed** and mapped to `extraction_failed`.
- **Empty candidate set avoids a wasted LLM round-trip** — `find_matching_premises` returns without calling `subprocess.run`.
- **CLI errors** use `SystemExit` (captured by `run_cli`) with a non-zero code and a human-readable message; test asserts `code == 1` and the word `"provide"`.

What is **not** covered:
- LLM subprocess failures (non-zero `returncode`, `stderr` content) — the mocks always use `returncode: 0`. Real-world `stderr`-based failures would need a new test.
- `search_fn` raising or returning non-JSON — it would propagate as an uncaught exception from `repair_smuggled_beliefs`, breaking the "captured, not raised" invariant. A test here would be valuable.
- Concurrent/same-`db_path` CLI invocations.

---

## Topics to Explore

- [file] `reasons/repair.py` — Implementation of the five functions exercised by this test; check whether `search_fn` failures are caught or propagated.
- [function] `reasons/repair.py:repair_smuggled_beliefs` — Verify the exact candidate-filter predicate and the `outlist` → `unless` projection.
- [file] `reasons/llm.py` — The transport abstraction whose `shutil.which`/`subprocess.run` calls are patched here; relevant for understanding the two-call ordering.
- [function] `reasons/api.py:add_justification` — Contract for `sl`, `unless`, and `db_path`; the "preserves outlist" test depends on it.
- [file] `reasons/cli.py` — Where `repair-smuggled` is registered; check the `--review-file` schema and output formatting ("REPAIRED", "dry run").
- [file] `tests/test_repair_premises.py` — A sibling feature; useful to diff how premise repair is scaffolded versus smuggled-premise repair.

---

## Beliefs

- `repair-processes-only-invalid-beliefs` — `repair_smuggled_beliefs` returns zero results when every entry in `review_results` has `valid=True`.
- `extraction-failure-short-circuits-search` — An empty `extract_smuggled_claim` response produces status `extraction_failed` and prevents the search/match LLM call from being made.
- `candidate-filtering-precedes-llm-matching` — Antecedent-duplicate and derived-belief candidates are removed before `find_matching_premises` runs, causing `no_candidates` without a second LLM round-trip.
- `dry-run-preserves-repaired-status` — `dry_run=True` still reports `status="repaired"` but never calls `reasons.api.add_justification`.
- `parse-match-response-filters-unknown-ids` — `parse_match_response` drops any `matched_ids` value that is not in the supplied candidate set, returning only the intersection.
- `outlist-propagates-to-unless` — The original `SL` justification's `outlist` is forwarded as the `unless` argument to `add_justification` (asserted as `"blocker-x"` in `test_preserves_outlist`).
- `repair-captures-per-belief-errors` — A review entry referencing a belief absent from `nodes` yields a result with `status="error"` rather than raising.
