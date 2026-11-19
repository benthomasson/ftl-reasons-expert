---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_derive.py

# Explanation of `tests/test_derive.py`

## Purpose

This is the **pytest test suite for `reasons.derive`**, the module responsible for LLM-assisted *reasoning-chain derivation* — i.e. proposing new derived beliefs from existing premises, validating them, applying them to the reason network, and orchestrating the end-to-end CLI workflow (including deduplication of near-duplicate "retracted" beliefs and cluster-based prompt budgeting).

The file's role is to lock down:

1. **Prompt construction** — filtering by depth, topic, premises-only, dependents, sampling, clustering, budget, custom templates.
2. **Proposal parsing** — both current and legacy (`v0.9`) markdown formats, `DERIVE`/`GATE` kinds, `ANY` mode, backtick stripping.
3. **Validation gates** — antecedent existence, minimum 2 antecedents, duplicate IDs, similarity to retracted beliefs.
4. **Application semantics** — how `derive` vs `gate`, and `all` vs `any` modes materialize into justifications / outlists / truth values.
5. **Round-trip persistence** — `write_proposals_file` → `parse_proposals` → `validate_proposals` → `apply_proposals`.
6. **Deduplication** — `_tokenize_id`, `_jaccard`, `find_similar_out`, `api.deduplicate`, and the plan-based workflow (`write_dedup_plan` / `parse_dedup_plan` / `apply_dedup_plan`).
7. **CLI reporting** — the `derive` command in `reasons.cli.main`, with mocked LLM subprocess calls, checks that a JSON report is written.
8. **Cluster integration** — conditional tests gated on `HAS_CLUSTER_DEPS`.

Everything is tested in a **white-box STYLE**: private helpers (`_detect_agents`, `_get_depth`, `_tokenize_id`, `_jaccard`, `_sample_beliefs`, `_filter_by_topic`, `_build_derived_section`) are imported and exercised directly, not just through public APIs.

---

## Key Components

### Fixtures

| Fixture | Purpose |
|---|---|
| `db(tmp_path)` | Creates `reason.db` under pytest's `tmp_path` and calls `api.init_db`. Every other fixture depends on it, so each test gets a fresh isolated DB. |
| `simple_network(db)` | 3 premises (`fact-a`, `fact-b`, `fact-c`) + 1 derived node `derived-ab` with `sl="fact-a,fact-b"`. Used by most build/apply tests. |
| `agent_network(db)` | Two namespaced "imported" agents (`agent-a:*`, `agent-b:*`) plus an `:active` premise for each, to exercise agent detection, cross-agent prompts, and `_filter_by_topic`. |

### Module under test — imported symbols

From `reasons.derive`:

| Symbol | Role | Contract (verified in tests) |
|---|---|---|
| `build_prompt(nodes, **opts) -> (prompt, stats)` | Assemble the LLM prompt from exported nodes. | Returns a `str` prompt plus a `stats` dict with keys like `total_in`, `total_derived`, `max_depth`, `agents`, `agent_names`, `min_depth`, `max_depth_filter`, `topic`, `budget`, `sample`, `cluster`, `n_clusters`, `embedding_model`, `intra_cluster`, `focus_cluster`. Supports `domain`, `min_depth`, `max_depth_filter`, `premises_only`, `has_dependents`, `topic`, `budget`, `sample`, `seed`, `cluster`, `intra_cluster`, `prompt_template`. Raises `ValueError` for unknown placeholders and malformed braces. |
| `parse_proposals(response) -> list[dict]` | Parse markdown `### DERIVE …` / `### GATE …` blocks. | Each returned dict has `kind` (`derive`/`gate`), `id`, `text`, `antecedents` (list), `unless` (list), `label`, `mode` (`all`/`any`, defaults to `all`). Strips backticks from IDs/lists. Handles legacy `### DERIVE:` and `**Antecedents**:` format. |
| `validate_proposals(proposals, nodes) -> (valid, skipped)` | Filter proposals against the current network. | Rejects with reason if: an antecedent id is missing, fewer than **2 antecedents**, the id already exists, or the id is *similar* to an OUT node (via `find_similar_out`). `skipped` items are `(proposal, reason_str)` tuples. |
| `apply_proposals(proposals, db_path) -> list[(proposal, result)]` | Add each proposal to the DB. | Returns list of `(proposal, result_dict)` where `result_dict["truth_value"]` is `"IN"`/`"OUT"`. `any` mode → multiple single-antecedent SL justifications; `gate` kind → adds `unless` as outlist. |
| `write_proposals_file(proposals, path)` | Serialize proposals to markdown that `parse_proposals` can read back. | Round-trip invariant enforced in `test_write_proposals_file_roundtrip`. |
| `find_similar_out(proposal_id, nodes) -> list[(node_id, score)]` | Find OUT nodes whose ID tokens are similar to `proposal_id`. | Only considers `truth_value == "OUT"`; uses `_jaccard` on `_tokenize_id` outputs; threshold **≥ 0.5**. |

### Private helpers (defined in `reasons.derive`, imported for testing)

- **`_detect_agents(nodes)`** → dict `{agent_name: [belief_ids…]}` — detects namespaced `agent-a:*` nodes and excludes the `:active` sentinel from the belief list.
- **`_get_depth(node_id, nodes, derived)`** → int — recursively computes derivation depth from justification antecedents (0 for premises).
- **`_filter_by_topic(nodes, topic)`** → dict — keeps nodes whose id **or** text matches **any** whitespace-separated keyword (OR semantics).
- **`_sample_beliefs(ids, budget, rng=None)`** → list — returns all ids if `budget >= len(ids)`, else samples `budget`; the optional `rng` gives reproducibility.
- **`_tokenize_id(id)`** → set of string tokens split on non-alphanumeric chars (so `agent-a:gl108-disabled` → `{"agent","a","gl108","disabled"}`).
- **`_jaccard(a, b)`** → float in `[0,1]`; empty vs non-empty = 0.0.
- **`_build_derived_section(nodes, derived, max_derived=N)`** → markdown string; when `len(derived) > max_derived`, emits `"<k> more derived conclusions omitted"`.

### CLI / mock helpers

- **`_run_cli(*args, db_path=None)`** — swaps `sys.argv`, `sys.stdout`, `sys.stderr` via `unittest.mock.patch`, calls `reasons.cli.main()`, recovers `SystemExit` code (default `0`), returns `(stdout, stderr, code)`.
- **`_mock_derive_response(proposals)`** — constructs a duck-typed subprocess result with `returncode`, `stdout` containing `### DERIVE …` blocks, and empty `stderr`.
- **`skip_no_cluster`** — `pytest.mark.skipif(not HAS_CLUSTER_DEPS, …)` gate for cluster-dependent tests.

---

## Patterns

1. **Layered test pyramid per helper.** Tests target the smallest unit first (`_jaccard`, `_tokenize_id`), then composed units (`find_similar_out`, `validate_proposals`), then integration (`apply_proposals` + `api.show_node` verification), then CLI (`_run_cli`).
2. **Fixture-based DB isolation.** Each fixture chains `tmp_path → db → {simple_network, agent_network}`, guaranteeing no cross-test state leakage.
3. **Round-trip testing.** `test_write_proposals_file_roundtrip` asserts `parse_proposals(write_proposals_file(p))` reconstructs the same shape — this forces the writer and parser to stay in sync.
4. **Mocked external process boundary.** LLM CLI invocations are stubbed at `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, **not** at `reasons.derive`, so the actual subprocess-handling code path is exercised.
5. **Antecedent state assertions via `api.show_node`.** Application tests don't just check `result["truth_value"]`; they re-read the DB to confirm the invariant (`fact-a` is in `justifications[0]["antecedents"]`, etc.).
6. **Negative-space assertions.** Tests like `assert "derived-ab" not in prompt` and `assert "gl108-safety-validation-disabled" not in node["justifications"][0]["outlist"]` catch over-permissive behavior, not just under-permissive ones.
7. **White-box private-function imports.** Uncommon in integration suites but necessary here because dedup/topic/depth logic is intentionally hidden behind the public `build_prompt`/`validate_proposals`.

---

## Dependencies

**Imports (what this file uses):**
- Standard library: `pathlib.Path`, `random`, `json`, `sys`, `io.StringIO`, `unittest.mock.patch`, `os`.
- Third-party: `pytest`.
- Project: `reasons.api` (`init_db`, `add_node`, `export_network`, `show_node`, `retract_node`, `get_status`, `deduplicate`, `write_dedup_plan`, `parse_dedup_plan`, `apply_dedup_plan`), `reasons.derive` (10 public + private helpers), `reasons.cli.main`, `reasons.cluster.HAS_CLUSTER_DEPS` and `cluster_beliefs_intra`.

**Imported by:** only the test runner — no production code imports this file. It is however the primary **regression net** for `reasons/derive.py`.

**Patched symbols:** `reasons.llm.shutil.which`, `reasons.llm.subprocess.run`, `sys.argv`, `sys.stdout`, `sys.stderr`. Not `reasons.derive` directly — the tests intentionally reach one layer deeper so mock drift is caught.

---

## Flow (per major test group)

- **Prompt-building path:** fixture → `api.export_network` → `build_prompt(nodes, **opts)` → inspect `prompt` string and/or `stats` dict. Each filter option is tested both alone (`test_build_prompt_min_depth`, `_max_depth`, `_premises_only`, `_has_dependents`, `_with_topic`, `_with_budget`, `_with_sample`) and in combination (`test_build_prompt_depth_range`).
- **Parse path:** literal markdown string → `parse_proposals` → assert dict shape. Multiple variants are tested (derive, gate, mode `ANY`, legacy `### DERIVE:`, backtick wrapping).
- **Validate path:** hand-built `nodes` dict + hand-built proposals → `validate_proposals` → assert split into `(valid, skipped)` and inspect skip-reason substrings.
- **Apply path:** `apply_proposals` → returns `(proposal, result)` list → verify `result["truth_value"]` → re-read via `api.show_node`. Gated and `any` cases additionally mutate the network (`api.retract_node`) and re-assert truth value transitions.
- **Dedup plan path:** `api.deduplicate(auto=False)` → `api.write_dedup_plan(clusters, path)` → file must contain `[KEEP]` and `[RETRACT]` markers → `api.parse_dedup_plan(text)` → `api.apply_dedup_plan(parsed, db_path)` → `retracted` populated, `errors` empty.
- **CLI/report path:** patch LLM subprocess → `_run_cli("derive", "--auto", "--report-dir", …)` → assert exit code 0 → `os.listdir(report_dir)` yields `derive-*.json` → load JSON and assert `status == "complete"`, `model == "claude"`, and round structure (`rounds`, `rounds[0]["proposals_found"]`, `rounds[0]["added"]`, `total_added`). `--exhaust` adds a second round that produces zero new proposals → `report["rounds"][1]["added"] == 0`.
- **Cluster path:** skipped when `HAS_CLUSTER_DEPS` is `False`; otherwise asserts `stats["cluster"] is True`, embedding model metadata, and that `cluster_beliefs_intra` rotates focus cluster between `round_num=0` and `round_num=1` with the same `seed`.

---

## Invariants

1. **Minimum antecedent arity** — `validate_proposals` rejects any proposal with `< 2` antecedents; skip reason contains the literal `"at least 2 antecedents"`.
2. **Antecedent existence** — every antecedent id must appear in `nodes`; skip reason contains the missing id.
3. **ID non-collision** — a proposal id equal to an existing node id is rejected with `"already exists"`.
4. **Retracted-similarity guard** — `find_similar_out` and the validation check only fire against nodes with `truth_value == "OUT"`; threshold via `_jaccard` is `≥ 0.5`. Skip reason contains `"similar to retracted"`.
5. **`_detect_agents` excludes `:active`** — a node named `agent-a:active` marks the agent as *detected* but is **not** counted as one of its beliefs (verified: `len(agents["agent-a"]) == 2` given three namespaced ids).
6. **`_get_depth` semantics** — `0` for premises, `1` for a node whose antecedents are all premises, `2` for nodes depending on a depth-1 node.
7. **`gate` semantics** — a gate whose `unless` antecedent is IN produces an `OUT` node; retracting the blocker flips it to `IN`.
8. **`any` mode semantics** — creates one justification per antecedent (each length-1), so retracting any one antecedent leaves the node IN as long as another remains.
9. **`all` mode is the default** — proposals lacking an explicit `- Mode:` line are treated as `mode == "all"`.
10. **Template placeholders** — only a whitelist (`total_in`, `max_depth`, `beliefs_section`, `derived_section`, `domain_context`, `cross_agent_task`, `agents_stats`, `total_derived`) is accepted; unknown names → `ValueError("…unknown placeholder…")`; unbalanced braces → `ValueError("…malformed braces…")`.
11. **Dedup rewrite-before-retract** — dependents' justification antecedents and outlists are rewritten to point at the `KEEP` node **before** the `RETRACT` nodes are removed; otherwise `safety-pipeline-broken` would lose IN status.
12. **Dedup tie-breaking** — `result["clusters"][0]["kept"]` is always excluded from `result["retracted"]`, and the kept node wins by **dependent count** (`gl108-validation-disabled` kept because it has 2 dependents vs 0).

---

## Error Handling

- **`build_prompt`** raises `ValueError` for unknown template placeholders and malformed brace strings. Tests use `pytest.raises(ValueError, match="…")` — this is a **fail-fast** API contract; callers must not pass arbitrary user-supplied templates without catching.
- **`api.apply_dedup_plan`** returns a `dict` with `errors: list[str]` — it does **not** raise. `test_apply_dedup_plan_missing_node` verifies a missing node produces one error entry with the id in the string.
- **CLI failures** propagate as `SystemExit` codes, captured by `_run_cli`; tests assert `code == 0` on successful paths.
- **Dedup with no clusters** is not an error: `result["clusters"] == []` and `result["retracted"] == []`.
- **Validation failures are soft** (returned in `skipped`, not raised) — this keeps the LLM-proposal loop non-fatal so a single bad proposal doesn't abort a batch.
- **Missing optional dependency** (`HAS_CLUSTER_DEPS == False`) is handled by `pytest.mark.skipif` rather than try/except inside production code.

---

## Topics to Explore

- [file] `reasons/derive.py` — The module under test; especially `build_prompt`, `validate_proposals`, and the dedup helpers to see the actual implementation shape behind the contract this suite pins.
- [function] `reasons/api.py:deduplicate` — Clustering algorithm that produces `clusters` with `kept`, `size`, and `dependents`; the tie-breaking rule is asserted in `test_deduplicate_auto_retracts` and `test_deduplicate_rewrites_dependents`.
- [function] `reasons/cli.py:main` — Dispatches `derive` subcommand; the `--auto`, `--exhaust`, `--report-dir`, `--no-report` flags flow through here and are exercised by `_run_cli`.
- [file] `reasons/llm.py` — Provides the `shutil.which` / `subprocess.run` boundary that the CLI tests patch; understanding the actual LLM invocation helps interpret `_mock_derive_response`.
- [function] `reasons/cluster.py:cluster_beliefs_intra` — Implements round-number rotation asserted by `test_intra_cluster_rotation`; worth reading to see how the `focus_cluster` selector advances.

---

## Beliefs

- `validate-proposals-requires-two-antecedents` — `validate_proposals` moves any proposal with fewer than 2 antecedents into `skipped` with a reason containing `"at least 2 antecedents"`.
- `gate-proposals-return-out-when-unless-in` — Applying a `kind == "gate"` proposal whose `unless` antecedent is IN yields `truth_value == "OUT"` until the blocker is retracted, at which point it becomes `"IN"`.
- `any-mode-creates-one-justification-per-antecedent` — Applying a proposal with `mode == "any"` produces N justifications of length 1 (one per antecedent), and retracting any single antecedent leaves the node IN.
- `find-similar-out-threshold-is-jaccard-half` — `find_similar_out` matches only nodes with `truth_value == "OUT"` whose `_tokenize_id` Jaccard similarity to the proposal id is ≥ 0.5.
- `dedup-rewrites-before-retracting` — Both `api.deduplicate(auto=True)` and `api.apply_dedup_plan` rewrite justification antecedents and outlist references to the KEEP node before retracting the duplicates, so dependents whose justifications referenced the retracted id stay IN.
- `derive-report-json-has-rounds` — The `derive` CLI with `--report-dir` writes a JSON file named `derive-<ts>.json` containing keys `status`, `model`, `exhaust`, `rounds`, and `total_added`; `--exhaust` appends a saturation round whose `added == 0`.
