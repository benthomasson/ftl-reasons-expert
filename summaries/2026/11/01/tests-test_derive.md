---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_derive.py

# `tests/test_derive.py` — Test Suite for Reasoning Chain Derivation

## Purpose

This file is the test suite for the `reasons.derive` module, which implements the core **reasoning chain derivation** pipeline in the `ftl-reasons` project. The pipeline's job is to take an existing belief network (premises + derived conclusions), feed a structured prompt to an LLM, parse the LLM's proposed new beliefs, validate them against the network, and apply them to the database.

The tests cover the **entire lifecycle**: prompt construction → LLM response parsing → validation → application → deduplication → reporting. They also test the **deduplicate** subsystem (finding near-duplicate beliefs via Jaccard similarity on tokenized IDs) and the **dedup plan** workflow (a human-in-the-loop approve/retract flow).

## Key Components

### Fixtures

| Fixture | Purpose |
|---------|---------|
| `db` | Creates a temp SQLite DB via `api.init_db`, yields the path |
| `simple_network` | 3 premises (`fact-a`, `fact-b`, `fact-c`) + 1 derived node (`derived-ab`) justified by `fact-a` and `fact-b` |
| `agent_network` | Two namespaced agents (`agent-a`, `agent-b`) each with an `:active` root and 1–2 belief nodes, simulating imported agent knowledge |

### Functions Under Test (from `reasons.derive`)

- **`build_prompt(nodes, ...) → (prompt: str, stats: dict)`** — Assembles the LLM prompt. Accepts filtering knobs: `domain`, `min_depth`, `max_depth_filter`, `premises_only`, `has_dependents`, `topic`, `budget`, `sample`, `seed`, `cluster`, `intra_cluster`, `prompt_template`. Returns a stats dict tracking counts, depths, agent info, and applied filters.

- **`parse_proposals(response: str) → list[dict]`** — Extracts structured proposals from LLM markdown output. Recognizes `### DERIVE <id>` and `### GATE <id>` blocks. Parses antecedents, unless (outlist), label, and optional `Mode: ANY` / `Mode: ALL`. Strips backticks from IDs. Defaults mode to `"all"`.

- **`validate_proposals(proposals, nodes) → (valid, skipped)`** — Returns two lists. A proposal is skipped if: (a) an antecedent ID doesn't exist in `nodes`, (b) the proposal's `id` already exists, or (c) the ID is "similar to retracted" (Jaccard ≥ threshold against an OUT node).

- **`apply_proposals(proposals, db_path) → list[(proposal, result_dict)]`** — Writes each proposal into the DB. For `kind="derive"`, creates a justified node. For `kind="gate"`, creates a node with an outlist. For `mode="any"`, creates one justification per antecedent (N separate single-antecedent justifications). Returns the computed `truth_value`.

- **`write_proposals_file(proposals, path)`** — Serializes proposals to a markdown file in the `### DERIVE` / `### GATE` format (round-trip compatible with `parse_proposals`).

- **`find_similar_out(candidate_id, nodes) → list[(node_id, score)]`** — Fuzzy-matches a candidate ID against OUT-status nodes using Jaccard similarity on tokenized IDs. Returns matches with score ≥ 0.5.

- **`_detect_agents(nodes) → dict[str, list[str]]`** — Scans node IDs for the `agent-X:...` namespace pattern. Returns `{agent_name: [belief_ids]}`, excluding `:active` sentinel nodes.

- **`_filter_by_topic(nodes, topic: str) → dict`** — Case-insensitive substring match against both node ID and text. Supports space-separated keywords with OR semantics.

- **`_sample_beliefs(ids, budget, rng=None) → list[str]`** — Returns all IDs if under budget; otherwise a reproducible random sample of size `budget`.

- **`_get_depth(node_id, nodes, derived) → int`** — Computes the longest derivation chain from premises (depth 0) to the given node.

- **`_tokenize_id(id: str) → set[str]`** — Splits on hyphens and colons, lowercases. E.g., `"agent-a:gl108-disabled"` → `{"agent", "a", "gl108", "disabled"}`.

- **`_jaccard(a: set, b: set) → float`** — Standard Jaccard index; returns 0.0 if either set is empty.

- **`_build_derived_section(nodes, derived, max_derived) → str`** — Renders derived beliefs as `####` headings. If count exceeds `max_derived`, caps output and appends a "N more derived conclusions omitted" note.

### CLI-level Tests (bottom of file)

These use `_run_cli` (a helper that patches `sys.argv`, `sys.stdout`, `sys.stderr`) and `unittest.mock.patch` to simulate the full `reasons derive --auto` command with a mocked LLM response. They verify:
- Report JSON is written to `--report-dir`
- `--no-report` suppresses report output
- `--exhaust` mode runs multiple rounds until saturation

## Patterns

1. **Round-trip testing**: `write_proposals_file` → `parse_proposals` must be identity (see `test_write_proposals_file_roundtrip`). Same pattern for `write_dedup_plan` → `parse_dedup_plan`.

2. **Mocked LLM boundary**: The LLM is never actually called. `_mock_derive_response` builds a fake `subprocess.CompletedProcess`-like object, and `patch("reasons.llm.subprocess.run")` intercepts the call. This keeps tests deterministic and fast.

3. **Conditional test skipping**: Cluster tests use `@pytest.mark.skipif(not HAS_CLUSTER_DEPS, ...)` so the suite passes in environments without `sentence-transformers`/`scikit-learn`.

4. **State-transition assertions**: Several tests verify causal chains — e.g., `test_apply_proposals_with_gate` asserts the gated belief is OUT, then retracts the blocker and asserts it flips to IN. `test_apply_proposals_any_mode` retracts one antecedent and verifies the belief survives.

5. **Dedup cluster invariant**: The "keep" node is determined by highest dependent count. Tests verify this (`test_deduplicate_auto_retracts` gives the kept node 2 dependents, retracted nodes 0).

6. **Separation of detection vs. action**: `deduplicate(auto=False)` only finds clusters; `apply_dedup_plan` performs retraction + rewrite. This enables the human-in-the-loop plan workflow.

## Dependencies

**Direct imports:**
- `reasons.api` — the public API (init_db, add_node, export_network, show_node, retract_node, deduplicate, write_dedup_plan, parse_dedup_plan, apply_dedup_plan, get_status)
- `reasons.derive` — the module under test (all 14 functions listed above)
- `reasons.cli` — the CLI entry point (`main`) for end-to-end tests
- `reasons.cluster` — optional, for embedding-based clustering tests
- `pytest`, `unittest.mock`, `pathlib`, `json`, `sys`, `io`, `os`, `random`

**Imported by:** No other test file imports from `test_derive.py` — it's a leaf test module. However, it exercises code paths that `reasons.derive` is imported by `reasons.cli` (the `derive` subcommand) and potentially `reasons.mcp_server.py`.

## Flow

The test file mirrors the **derive pipeline** in execution order:

```
1. build_prompt(nodes, filters...) → (prompt, stats)
       │
       ▼  [LLM call — mocked in tests]
       │
2. parse_proposals(llm_response) → [proposal dicts]
       │
       ▼
3. validate_proposals(proposals, nodes) → (valid, skipped)
       │
       ▼
4. apply_proposals(valid, db_path) → [(proposal, result)]
       │
       ▼
5. (Optional) write_proposals_file → parse → apply  [accept workflow]
       │
       ▼
6. deduplicate(auto) → clusters → write_dedup_plan → parse → apply_dedup_plan
```

Each test exercises one stage or a specific edge case within a stage. The CLI tests at the bottom exercise the full chain 1→4→report.

## Invariants

1. **Antecedent existence**: Every antecedent in a valid proposal must reference a node present in the network. `validate_proposals` enforces this.
2. **No duplicate IDs**: A proposal whose `id` matches an existing node is rejected with "already exists".
3. **Similar-to-retracted guard**: A proposal whose ID has Jaccard ≥ 0.5 with any OUT node is rejected. This prevents the LLM from "re-deriving" a belief that was deliberately retracted under a slightly different name.
4. **Mode semantics**: `mode="all"` (default) means all antecedents must be IN for the derived belief to be IN. `mode="any"` means at least one antecedent being IN suffices (implemented as N separate single-antecedent justifications).
5. **Gate semantics**: A gated belief is IN iff all antecedents are IN AND no unless/outlist node is IN. Retraction of an outlist node restores the belief to IN.
6. **Budget cap**: `_build_derived_section` never renders more than `max_derived` entries; overflow is summarized.
7. **Dedup keep-selection**: The node with the most dependents is kept; the rest are retracted. If a retracted node has dependents, their justifications are **rewritten** to point at the kept node.
8. **Prompt template placeholders**: Only a fixed set of `{placeholders}` is allowed. Unknown placeholders raise `ValueError("unknown placeholder")`. Malformed braces (unclosed) raise `ValueError("malformed braces")`.

## Error Handling

- **`validate_proposals`** never raises; it partitions into `(valid, skipped)` where each skipped entry is `(proposal, reason_string)`. Callers inspect the reason to decide whether to surface to the user.
- **`apply_dedup_plan`** returns a dict with `"errors"` list (e.g., `"nonexistent"` if a node in the plan is missing from the DB) alongside `"retracted"`. It doesn't raise for missing nodes — it's a soft-fail.
- **`build_prompt`** raises `ValueError` for bad template placeholders or malformed braces. This is a programmer error, not a runtime data error.
- **CLI tests** capture `SystemExit` to get the exit code. A non-zero code indicates failure; the test asserts `code == 0` for success paths.
- **Cluster tests** are silently skipped (not failed) when optional dependencies are absent, via `pytest.mark.skipif`.

---

## Topics to Explore

- [file] `reasons/derive.py` — The production module under test; contains the full derive pipeline, prompt templates, and the Jaccard-based dedup logic. Understanding its internal structure is the natural next step after reading the tests.
- [function] `reasons/derive.py:validate_proposals` — The gatekeeper that enforces antecedent existence, duplicate ID rejection, and the "similar to retracted" guard. Its interaction with `find_similar_out` and `_tokenize_id` is the most subtle validation logic in the project.
- [file] `tests/test_derive_budget.py` — A companion test file focused specifically on budget/sampling edge cases. Comparing it with the budget tests here reveals whether additional coverage was needed.
- [file] `reasons/cluster.py` — Implements embedding-based clustering (`cluster_beliefs_intra`) used by `build_prompt(cluster=True)`. The conditional import and skip pattern in the test file makes this the boundary for optional ML dependencies.
- [general] `dedup-plan-workflow` — The human-in-the-loop dedup flow (detect → write plan → user edits KEEP/RETRACT → apply) touches `api.deduplicate`, `api.write_dedup_plan`, `api.parse_dedup_plan`, and `api.apply_dedup_plan`. Tracing how dependent-rewriting works across these functions reveals the most complex state-mutation logic in the codebase.

## Beliefs

- `validate-proposals-rejects-similar-retracted` — A proposal whose ID has Jaccard similarity ≥ 0.5 with any OUT-status node is rejected even if its ID doesn't exactly match, preventing re-derivation of deliberately retracted beliefs under variant names.
- `mode-any-produces-n-justifications` — When `mode="any"`, `apply_proposals` creates N separate single-antecedent justifications (one per antecedent) rather than one multi-antecedent justification, so the belief stays IN as long as any single antecedent remains IN.
- `dedup-keep-is-most-dependent` — In `apply_dedup_plan`, the kept node is the one with the highest dependent count; all other cluster members are retracted, and any derived beliefs depending on a retracted member have their justifications rewritten to reference the kept node.
- `build-prompt-raises-on-unknown-placeholder` — `build_prompt` validates `prompt_template` at call time and raises `ValueError` (not a silent skip) if it contains an unrecognized `{placeholder}` or malformed braces, making template errors fail-fast rather than producing a garbled LLM prompt.
- `parse-proposals-defaults-mode-to-all` — When a proposal block omits the `Mode:` field, `parse_proposals` sets `mode` to `"all"`, meaning all antecedents must be IN for the derived conclusion to hold; `Mode: ANY` must be explicitly stated.
