# File: tests/test_derive.py

**Date:** 2026-05-08
**Time:** 14:14

# `tests/test_derive.py` — Tests for Reasoning Chain Derivation

## Purpose

This file is the test suite for the **derive** subsystem — the part of `reasons_lib` that uses an LLM to propose new derived beliefs from existing ones in the belief network. It tests the full pipeline: prompt construction, LLM response parsing, proposal validation, application to the database, duplicate detection, deduplication workflows, and CLI report generation.

It's the most comprehensive test file in the project, covering not just `reasons_lib.derive` but also `api.deduplicate`, `api.write_dedup_plan`, `api.parse_dedup_plan`, and `api.apply_dedup_plan` — all of which are downstream of the derive workflow.

## Key Components

### Fixtures

- **`db(tmp_path)`** — Creates a fresh SQLite database per test via `api.init_db`. Every test gets an isolated belief network.
- **`simple_network(db)`** — Four nodes: three premises (`fact-a`, `fact-b`, `fact-c`) and one derived node (`derived-ab`) with an SL justification depending on `fact-a` and `fact-b`. This is the workhorse fixture for most tests.
- **`agent_network(db)`** — Simulates multi-agent federation with colon-namespaced nodes (`agent-a:knows-auth`, `agent-b:knows-gateway`). Tests cross-agent derivation prompting.

### Test Groups (in order)

1. **Prompt construction** (`test_build_prompt_*`) — Verifies `build_prompt` generates correct prompts with proper stats: total IN/derived counts, depth calculations, agent detection, domain injection, filtering by depth range, premises-only mode, has-dependents mode, topic filtering, budget limits, and sampling.

2. **Internal helpers** (`test_detect_agents`, `test_get_depth`, `test_filter_by_topic_*`, `test_sample_beliefs_*`) — Unit tests for the private functions that `build_prompt` delegates to.

3. **Proposal parsing** (`test_parse_proposals_*`) — Validates `parse_proposals` against multiple markdown formats: DERIVE blocks, GATE blocks (with `unless` outlists), multiple proposals, backtick-wrapped IDs, and the older v0.9 format with bold field names.

4. **Proposal validation** (`test_validate_proposals_*`) — Tests `validate_proposals` catches missing antecedents, already-existing nodes, and (critically) proposals whose IDs are Jaccard-similar to retracted (OUT) beliefs.

5. **Proposal application** (`test_apply_proposals*`) — Tests `apply_proposals` actually creates nodes in the database with correct truth values, including GATE beliefs that start OUT when their outlist condition is IN and flip IN when it's retracted.

6. **Duplicate detection** (`test_tokenize_id`, `test_jaccard_*`, `test_find_similar_out_*`) — Unit tests for the fuzzy-match machinery that prevents re-deriving retracted beliefs under variant IDs.

7. **Deduplication** (`test_deduplicate_*`, `test_write_and_parse_dedup_plan`, `test_dedup_plan_*`, `test_apply_dedup_plan_*`) — Tests the full dedup lifecycle: finding clusters of similar belief IDs, auto-retracting duplicates while keeping the one with the most dependents, rewriting justification antecedents and outlists to point at the kept belief, plan file round-tripping, and user-editable plans.

8. **CLI derive reports** (`test_derive_report_*`) — Integration tests for the `reasons derive` CLI command with `--auto`, `--exhaust`, `--report-dir`, and `--no-report` flags. Uses `_mock_derive_response` to simulate LLM output and verifies JSON report structure.

9. **Cluster integration** (`test_build_prompt_with_cluster*`) — Conditionally skipped tests that require `sentence-transformers` and `scikit-learn`. Tests embedding-based clustering for prompt construction.

### Helper Functions

- **`_run_cli(*args, db_path)`** — Captures stdout/stderr from `reasons_lib.cli.main` by patching `sys.argv` and streams. Returns `(stdout, stderr, exit_code)`.
- **`_mock_derive_response(proposals)`** — Builds a fake subprocess result object with DERIVE blocks formatted as the LLM would produce them.

## Patterns

**Fixture composition**: `simple_network` depends on `db`, which depends on `tmp_path`. Each test gets a fresh database — no cross-test contamination.

**Round-trip testing**: Several tests write data to a file, parse it back, and verify fidelity — e.g., `test_write_proposals_file_roundtrip` and `test_write_and_parse_dedup_plan`. This catches format drift between writers and parsers.

**CLI-as-black-box**: The derive report tests invoke the full CLI entry point with mocked LLM subprocess calls, testing the integration between argument parsing, derivation logic, and report writing.

**Conditional skip**: The cluster tests use `pytest.mark.skipif` on `HAS_CLUSTER_DEPS` to gracefully degrade when heavy ML dependencies aren't installed.

**Truth value as behavioral assertion**: GATE belief tests assert truth values change when outlist conditions are retracted — they're testing the TMS propagation semantics, not just CRUD.

## Dependencies

**Imports from project:**
- `reasons_lib.api` — The public API for all belief operations (add, retract, export, show, deduplicate)
- `reasons_lib.derive` — The module under test (prompt building, parsing, validation, application)
- `reasons_lib.cli.main` — CLI entry point for integration tests
- `reasons_lib.cluster.HAS_CLUSTER_DEPS` — Feature flag for optional ML dependencies

**External:**
- `pytest` — Test framework and fixtures
- `unittest.mock.patch` — Mocks for LLM subprocess calls and CLI streams
- `json`, `sys`, `io.StringIO`, `pathlib.Path`, `os`, `random` — Standard library utilities

**Nothing imports this file** — it's a leaf test module.

## Flow

A typical derive workflow tested here:

1. `build_prompt(nodes, ...)` → constructs an LLM prompt with filtered/sampled beliefs and stats
2. LLM returns markdown with `### DERIVE` / `### GATE` blocks
3. `parse_proposals(response)` → extracts structured proposal dicts
4. `validate_proposals(proposals, nodes)` → filters out invalid (missing antecedents, duplicates, similar-to-retracted)
5. `apply_proposals(valid, db_path)` → creates nodes in the database, returns truth values

The dedup workflow is orthogonal:

1. `api.deduplicate(db_path)` → finds clusters of Jaccard-similar belief IDs
2. `api.write_dedup_plan(clusters, path)` → writes a human-editable markdown plan
3. User edits the plan (changing KEEP/RETRACT assignments)
4. `api.parse_dedup_plan(text)` → parses the edited plan
5. `api.apply_dedup_plan(plan, db_path)` → retracts duplicates, rewrites justifications

## Invariants

- **Antecedent existence**: `validate_proposals` rejects any proposal whose antecedents don't exist in the network.
- **No duplicate IDs**: Proposals with IDs already in the network are skipped.
- **No zombie re-derivation**: Proposals whose IDs are Jaccard-similar (≥0.5) to OUT beliefs are rejected with "similar to retracted" — this is the core bug that `find_similar_out` was built to prevent.
- **Dedup keeps most-connected**: Auto-dedup retains the belief with the most dependents and retracts the rest.
- **Justification rewrite on dedup**: When a duplicate is retracted, all justifications (antecedents and outlists) referencing it are rewritten to point at the kept belief, preserving dependent truth values.
- **GATE semantics**: A GATE belief is OUT when its outlist node is IN, and flips to IN when the outlist node is retracted.
- **Round-trip fidelity**: `write_proposals_file` output must parse identically through `parse_proposals`, and `write_dedup_plan` output must parse through `parse_dedup_plan`.

## Error Handling

- `validate_proposals` returns `(valid, skipped)` — skipped entries carry reason strings rather than raising exceptions. Tests assert on the reason text (e.g., `"similar to retracted"`, `"already exists"`).
- `apply_dedup_plan` returns `{"retracted": [...], "errors": [...]}` — missing nodes produce error strings rather than exceptions (`test_apply_dedup_plan_missing_node`).
- CLI tests capture `SystemExit` codes — the CLI communicates errors via exit codes and stderr, not exceptions.
- The `_run_cli` helper catches `SystemExit` and returns the code, so tests can assert on success (`code == 0`) without the exception bubbling up.

## Topics to Explore

- [file] `reasons_lib/derive.py` — The implementation behind `build_prompt`, `parse_proposals`, `validate_proposals`, and `apply_proposals` — understanding the prompt template and parsing regex is essential
- [function] `reasons_lib/api.py:deduplicate` — The dedup algorithm: how clusters are found, how the "keep" node is chosen, and how justification rewriting works
- [function] `reasons_lib/derive.py:find_similar_out` — The Jaccard similarity guard against re-deriving retracted beliefs under variant IDs
- [file] `tests/test_derive_budget.py` — Budget-specific edge cases that complement the budget tests here
- [general] `gate-belief-propagation` — How GATE beliefs interact with the TMS outlist mechanism and why outlist nodes aren't tracked in the dependents index (the known issue in CLAUDE.md)

## Beliefs

- `validate-proposals-rejects-jaccard-similar-to-out` — `validate_proposals` skips any proposal whose ID has Jaccard similarity ≥0.5 to an existing OUT belief, preventing re-derivation of retracted claims under variant names
- `dedup-auto-keeps-most-dependents` — `api.deduplicate(auto=True)` retains the cluster member with the most dependents and retracts all others
- `dedup-rewrites-justifications-on-retract` — When a duplicate is retracted, all antecedent and outlist references to it in other nodes' justifications are rewritten to point at the kept belief
- `gate-belief-out-when-outlist-in` — A GATE belief (created with `unless`) has truth value OUT when its outlist condition is IN, and flips to IN when the outlist node is retracted
- `derive-report-json-has-rounds-array` — The `reasons derive --auto --report-dir` command writes a JSON report with a `rounds` array, each entry containing `proposals_found` and `added` counts

