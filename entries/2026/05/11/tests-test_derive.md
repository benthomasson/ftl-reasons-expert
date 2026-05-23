# File: tests/test_derive.py

**Date:** 2026-05-11
**Time:** 13:02

# `tests/test_derive.py` — Test Suite for Reasoning Chain Derivation

## Purpose

This file is the primary test suite for the `derive` module (`reasons_lib/derive.py`), which is responsible for LLM-driven belief derivation — taking existing beliefs in a TMS network and asking an LLM to propose new derived beliefs or gated beliefs from them. It also tests the deduplication pipeline (`api.deduplicate`, `api.write_dedup_plan`, `api.apply_dedup_plan`) which detects and merges near-duplicate beliefs based on Jaccard similarity of their tokenized IDs.

The file owns coverage for the full derive lifecycle: prompt construction, LLM response parsing, proposal validation, proposal application, file-based round-tripping, duplicate detection, dedup plan workflows, and CLI report generation.

## Key Components

### Fixtures

- **`db(tmp_path)`** — Creates an ephemeral SQLite-backed reasons database per test. Every test gets a clean, isolated store.
- **`simple_network(db)`** — Seeds 3 premises (`fact-a`, `fact-b`, `fact-c`) and 1 derived node (`derived-ab` depending on `fact-a` + `fact-b`). This is the workhorse fixture for most prompt-building and apply tests.
- **`agent_network(db)`** — Seeds a multi-agent network with colon-namespaced IDs (`agent-a:knows-auth`, `agent-b:knows-gateway`). Tests cross-agent derivation features.

### Test Groups

**Prompt construction** (`test_build_prompt_*`): Verifies that `build_prompt` correctly assembles LLM prompts from a node dict, with stats tracking (total IN, derived count, max depth, agent count). Tests depth filtering (`min_depth`, `max_depth_filter`), premises-only mode, dependent filtering, domain annotation, topic filtering, budget limits, sampling, clustering, and custom prompt templates.

**Agent detection** (`test_detect_agents`): Validates `_detect_agents` identifies colon-namespaced agent prefixes and excludes `:active` sentinel nodes from belief counts.

**Depth calculation** (`test_get_depth`): Verifies `_get_depth` computes derivation chain depth — premises are 0, a node derived from premises is 1, a node derived from depth-1 nodes is 2.

**Parsing** (`test_parse_proposals_*`): Tests the markdown parser `parse_proposals` against multiple formats — `### DERIVE id`, `### GATE id`, old v0.9 format with backtick IDs and bold field names, and multi-proposal responses. Each proposal must yield `id`, `kind`, `antecedents`, `unless`, and `label`.

**Validation** (`test_validate_proposals_*`): Checks that `validate_proposals` rejects proposals with missing antecedents, duplicate IDs, and — critically — proposals whose IDs are Jaccard-similar to retracted (OUT) beliefs. This last check prevents the LLM from re-deriving beliefs that were previously retracted under slightly different names.

**Application** (`test_apply_proposals*`): Confirms `apply_proposals` creates real nodes in the database with correct truth values. For GATE proposals, verifies the outlist mechanism: a gated belief is OUT when its outlist node is IN, and flips to IN when the outlist node is retracted.

**Topic filtering** (`test_filter_by_topic_*`): Tests `_filter_by_topic` which matches keywords against both node IDs and text fields using OR semantics across space-separated keywords.

**Sampling** (`test_sample_beliefs_*`): Validates `_sample_beliefs` returns all beliefs when under budget, caps at budget when over, and is reproducible with a seeded RNG.

**Round-trip** (`test_write_proposals_file_roundtrip`, `test_accept_applies_proposals`): Proves the write-parse cycle is lossless — `write_proposals_file` produces markdown that `parse_proposals` reconstructs identically.

**Duplicate detection** (`test_tokenize_id`, `test_jaccard_*`, `test_find_similar_out_*`): Tests the Jaccard-similarity pipeline. `_tokenize_id` splits on hyphens and colons. `_jaccard` computes set overlap. `find_similar_out` finds OUT beliefs with >= 0.5 Jaccard similarity to a candidate ID.

**Deduplication** (`test_deduplicate_*`, `test_dedup_plan_*`, `test_apply_dedup_plan_*`): Full integration tests for the dedup pipeline. Covers cluster detection, auto-retraction, justification rewriting (antecedent and outlist references to retracted duplicates are rewritten to point at the kept belief), plan file round-tripping, user editability of plans, and error handling for missing nodes.

**CLI reports** (`test_derive_report_*`): Tests that `derive --auto` writes JSON reports with round-by-round stats, `--no-report` suppresses them, and `--exhaust` iterates until the LLM produces no new proposals.

## Patterns

- **Isolated fixture composition**: `simple_network` and `agent_network` build on `db`, keeping network setup separate from database lifecycle.
- **CLI testing via stdout capture**: `_run_cli` patches `sys.argv`, `sys.stdout`, and `sys.stderr` to test the CLI as a black box, catching `SystemExit` for return codes.
- **Mock LLM responses**: `_mock_derive_response` builds fake LLM output matching the expected markdown format, avoiding real API calls. The mock returns a duck-typed object with `returncode`, `stdout`, `stderr`.
- **Conditional skip**: Cluster tests use `pytest.mark.skipif` with the `HAS_CLUSTER_DEPS` flag to skip when `sentence-transformers` / `scikit-learn` aren't installed.
- **Round-trip testing**: Multiple tests write to a file and parse it back, ensuring serialization formats are self-consistent.

## Dependencies

**Imports from project:**
- `reasons_lib.api` — database init, node CRUD, export, deduplicate, dedup plan operations
- `reasons_lib.derive` — `build_prompt`, `parse_proposals`, `validate_proposals`, `apply_proposals`, `write_proposals_file`, `find_similar_out`, and private helpers (`_detect_agents`, `_filter_by_topic`, `_sample_beliefs`, `_get_depth`, `_tokenize_id`, `_jaccard`)
- `reasons_lib.cli.main` — CLI entrypoint for report tests
- `reasons_lib.cluster.HAS_CLUSTER_DEPS` — feature flag for optional ML dependencies

**External:** `pytest`, `unittest.mock.patch`, `json`, `pathlib`, `io.StringIO`

**Nothing imports this file** — it's a leaf test module.

## Flow

A typical derive cycle tested here follows this pipeline:

1. **Build prompt** — `build_prompt(nodes)` filters/samples beliefs, computes depth stats, detects agents, assembles a markdown prompt for the LLM.
2. **LLM call** (mocked) — Returns markdown with `### DERIVE` / `### GATE` blocks.
3. **Parse** — `parse_proposals(response)` extracts structured proposals from markdown.
4. **Validate** — `validate_proposals(proposals, nodes)` checks antecedent existence, ID uniqueness, and Jaccard-similarity to retracted beliefs. Returns `(valid, skipped)`.
5. **Apply** — `apply_proposals(valid, db_path)` creates nodes in the database via `api.add_node`.
6. **Report** (optional) — CLI writes a JSON report with per-round stats.

The dedup pipeline is parallel: `deduplicate()` → `write_dedup_plan()` → user review → `parse_dedup_plan()` → `apply_dedup_plan()`.

## Invariants

- **Antecedent existence**: `validate_proposals` rejects any proposal referencing a node not in the current network.
- **No duplicate IDs**: A proposal with an ID matching an existing node is skipped with "already exists".
- **Jaccard dedup guard**: Proposals with IDs >= 0.5 Jaccard similarity to any OUT belief are skipped as "similar to retracted". This prevents re-derivation of retracted beliefs under variant names.
- **GATE truth semantics**: A gated belief is OUT when any outlist node is IN; it flips to IN when all outlist nodes go OUT.
- **Dedup keeps the most-connected node**: When `auto=True`, the node with the most dependents survives; the rest are retracted.
- **Justification rewriting**: On dedup, all antecedent and outlist references to retracted nodes are rewritten to point at the kept node, so derived beliefs survive the retraction.
- **Round-trip fidelity**: `write_proposals_file` output must parse back to identical proposals via `parse_proposals`.
- **Custom template validation**: `build_prompt` raises `ValueError("unknown placeholder")` for unrecognized `{fields}` and `ValueError("malformed braces")` for unclosed braces.

## Error Handling

- `validate_proposals` does not raise — it partitions into `(valid, skipped)` tuples where each skip carries a human-readable reason string.
- `apply_dedup_plan` collects errors into `result["errors"]` rather than raising, allowing partial application (e.g., one missing node doesn't block the rest).
- `build_prompt` raises `ValueError` for bad custom templates — this is tested explicitly.
- CLI tests capture `SystemExit` and check the exit code, treating non-zero as failure.
- `find_similar_out` returns an empty list (not an error) when no matches exist.

## Topics to Explore

- [file] `reasons_lib/derive.py` — The implementation behind all tested functions; understanding the prompt template and parsing regex is essential
- [function] `reasons_lib/api.py:deduplicate` — The dedup engine including cluster detection, keeper selection, and justification rewriting
- [function] `reasons_lib/derive.py:build_prompt` — Prompt assembly with depth filtering, topic matching, agent grouping, budget sampling, and clustering
- [file] `reasons_lib/network.py` — The TMS core that `apply_proposals` delegates to for truth value propagation and outlist evaluation
- [general] `jaccard-dedup-threshold` — The 0.5 similarity threshold is hardcoded; understanding why this value was chosen and when it might produce false positives/negatives

## Beliefs

- `validate-proposals-rejects-jaccard-similar-to-out` — `validate_proposals` skips any proposal whose tokenized ID has >= 0.5 Jaccard similarity to an existing OUT belief, returning "similar to retracted" in the skip reason
- `dedup-rewrites-justifications-before-retract` — `api.deduplicate(auto=True)` rewrites antecedent and outlist references in dependent beliefs to point at the kept node before retracting duplicates, so derived beliefs survive
- `gate-truth-value-inverts-on-outlist-retraction` — A GATE belief applied with an IN outlist node starts OUT, and automatically flips to IN when that outlist node is retracted
- `parse-proposals-handles-two-format-versions` — `parse_proposals` accepts both the current format (`### DERIVE id`) and the v0.9 format (`### DERIVE: \`id\`` with bold field names and backtick-wrapped values)
- `build-prompt-raises-on-invalid-template-placeholders` — `build_prompt` raises `ValueError` with "unknown placeholder" for unrecognized `{fields}` and "malformed braces" for unclosed braces in custom templates

