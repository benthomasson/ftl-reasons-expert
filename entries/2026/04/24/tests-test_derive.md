# File: tests/test_derive.py

**Date:** 2026-04-24
**Time:** 16:42

## `tests/test_derive.py` — Explanation

### Purpose

This file is the test suite for `reasons_lib/derive.py`, the module responsible for **LLM-assisted reasoning chain derivation**. The derive module builds prompts that an LLM can use to propose new beliefs from existing ones, then parses, validates, and applies those proposals back into the belief network. This test file verifies every stage of that pipeline — from prompt construction through proposal round-tripping — plus a deduplication system that detects and merges near-duplicate beliefs.

It owns testing of two distinct capabilities:
1. **Derive pipeline**: `build_prompt` → LLM response → `parse_proposals` → `validate_proposals` → `apply_proposals`
2. **Deduplication**: `deduplicate` → `write_dedup_plan` → `parse_dedup_plan` → `apply_dedup_plan`

---

### Key Components

**Fixtures** (lines 25–55):
- `db` — creates a fresh SQLite database per test via `api.init_db`
- `simple_network` — 3 premises (`fact-a`, `fact-b`, `fact-c`) + 1 derived node (`derived-ab`). Used by most prompt and validation tests
- `agent_network` — simulates multi-agent imports with namespaced IDs like `agent-a:knows-auth`. Tests cross-agent prompt generation

**Prompt construction tests** (lines 58–144):
Tests for `build_prompt()` covering every filtering dimension:
- Basic stats computation (`total_in`, `total_derived`, `max_depth`)
- Domain context injection
- Agent detection and cross-agent prompt sections
- Depth filters: `min_depth`, `max_depth_filter`, depth range
- `premises_only` and `has_dependents` filters
- Topic keyword filtering
- Budget-constrained output
- Reproducible sampling via seeded RNG

**Parsing tests** (lines 148–226):
Tests for `parse_proposals()` validating two format versions:
- New format (v0.10+): `### DERIVE belief-id`
- Old format (v0.9): `### DERIVE: \`belief-id\`` with bold field names
- Backtick stripping on IDs and antecedents
- DERIVE vs GATE (outlist) distinction
- Multiple proposals in one response

**Validation tests** (lines 229–314):
Tests for `validate_proposals()` checking rejection of:
- Proposals with missing antecedents
- Proposals whose ID already exists in the network
- Proposals whose ID is similar to an OUT (retracted) belief — the **duplicate detection** guard

**Apply tests** (lines 252–281):
Tests for `apply_proposals()` verifying:
- Derived beliefs get truth value IN
- Gated beliefs respect outlist semantics (OUT when blocker is IN, IN when blocker is retracted)

**Helper unit tests**:
- `_tokenize_id` — kebab-case + namespace splitting into token sets
- `_jaccard` — Jaccard similarity: identical, disjoint, partial, empty-set edge case
- `_filter_by_topic` — keyword OR-matching on ID and text
- `_sample_beliefs` — budget-capped sampling with deterministic RNG
- `_get_depth` — recursive depth computation with memoization
- `_detect_agents` — namespace detection, excluding `:active` sentinels

**Deduplication tests** (lines 368–end):
Tests for the full dedup lifecycle via `api.deduplicate()`, `api.write_dedup_plan()`, `api.parse_dedup_plan()`, `api.apply_dedup_plan()`:
- Cluster detection by Jaccard similarity on IDs
- Auto-retraction mode (keeps the node with most dependents)
- Justification rewriting: when a duplicate is retracted, derived beliefs that depended on it get their antecedents rewritten to point at the kept node
- Outlist rewriting: same rewriting for outlist references
- Plan file round-trip: write → parse → apply
- User-editable plans: the user can swap which node is KEEP vs RETRACT
- Error handling for missing nodes in plans

---

### Patterns

1. **Fixture composition**: `simple_network` and `agent_network` build on `db`, creating a layered setup pattern. Each fixture returns `db_path` so tests chain through the same database.

2. **Pipeline testing**: Tests mirror the actual pipeline stages — build → parse → validate → apply — and several tests (`test_accept_applies_proposals`, `test_dedup_plan_end_to_end`) exercise the full pipeline end-to-end.

3. **Round-trip verification**: `test_write_proposals_file_roundtrip` and `test_write_and_parse_dedup_plan` verify that the serialization format is self-consistent (write → read → identical data).

4. **Behavioral verification through retraction**: `test_apply_proposals_with_gate` and `test_deduplicate_rewrites_dependents` don't just check initial state — they retract a node and verify the cascade produces the correct secondary effect.

5. **Internal helper exposure**: Private functions (`_detect_agents`, `_filter_by_topic`, etc.) are imported and tested directly. This is deliberate — the helpers encode critical logic (similarity matching, depth computation) that is worth testing in isolation.

---

### Dependencies

**Imports:**
- `reasons_lib.api` — the functional API layer that wraps the network/storage. Used for all database operations (`init_db`, `add_node`, `show_node`, `export_network`, `retract_node`, `deduplicate`, etc.)
- `reasons_lib.derive` — the module under test (prompt building, parsing, validation, application)
- `pathlib.Path` — file round-trip tests
- `pytest` — fixtures and assertions
- `random` — seeded RNG in sampling tests

**Imported by:** Nothing — this is a leaf test module.

---

### Flow

The test file validates this data flow:

```
Network (SQLite) → api.export_network() → nodes dict
    → build_prompt(nodes, filters...) → (prompt_text, stats)
    → [LLM generates response text]
    → parse_proposals(response) → list of proposal dicts
    → validate_proposals(proposals, nodes) → (valid, skipped)
    → apply_proposals(valid, db_path) → results
    → api.show_node() → verify IN/OUT
```

And the parallel dedup flow:

```
Network → api.deduplicate() → clusters
    → api.write_dedup_plan(clusters) → markdown file
    → api.parse_dedup_plan(text) → plan dicts
    → api.apply_dedup_plan(plan) → retraction + rewrite results
```

---

### Invariants

1. **Antecedent existence**: `validate_proposals` rejects any proposal whose antecedents or unless-list reference nodes not in the network.
2. **No duplicate IDs**: A proposal with an ID matching an existing node is skipped.
3. **No resurrection of retracted beliefs**: If a proposal's ID is Jaccard-similar (≥0.5) to any OUT node, it is skipped as "similar to retracted." This prevents LLMs from re-deriving beliefs that were intentionally retracted under slightly different names.
4. **Outlist semantics**: A GATE belief is OUT when its outlist nodes are IN. Retracting the blocker flips the gated belief to IN.
5. **Dedup rewriting preserves truth**: When a duplicate is retracted, its dependents survive by rewriting justifications to reference the kept node. The dependent's truth value stays IN.
6. **Dedup keeps the most-connected node**: Auto-mode retains the node with the most dependents.
7. **Plans are user-editable**: The dedup plan format supports users swapping KEEP/RETRACT markers before applying.

---

### Error Handling

- `validate_proposals` returns a `(valid, skipped)` tuple — errors are accumulated, not raised. Each skipped entry includes a reason string (e.g., `"missing nodes: [...]"`, `"already exists"`, `"similar to retracted belief: ..."`).
- `apply_proposals` catches exceptions from `api.add_node` and returns the error string alongside the proposal, rather than aborting the batch.
- `apply_dedup_plan` collects errors for missing nodes into `result["errors"]` instead of raising.
- No tests verify exception-raising paths — the derive module is designed to be error-accumulating, not exception-throwing.

---

## Topics to Explore

- [file] `reasons_lib/derive.py` — The implementation under test; contains the LLM prompt template, regex parsers, and the full derive pipeline
- [function] `reasons_lib/api.py:deduplicate` — The API-level deduplication logic including cluster detection, auto-retraction, and justification rewriting
- [file] `tests/test_derive_budget.py` — Companion test file focused specifically on budget/token constraints in prompt construction
- [general] `jaccard-similarity-threshold` — The 0.5 threshold in `find_similar_out` is a critical tuning parameter; too low allows false positives, too high misses variant rederivations
- [function] `reasons_lib/network.py:propagate` — The BFS propagation engine that makes retraction cascades and gated belief flips work correctly after `apply_proposals`

---

## Beliefs

- `derive-validate-rejects-similar-to-out` — `validate_proposals` rejects any proposal whose ID has ≥0.5 Jaccard similarity to an existing OUT node, preventing rederivation of retracted beliefs under variant names
- `derive-parse-supports-two-formats` — `parse_proposals` tries the new format first (v0.10+: `### DERIVE id`), falls back to old format (v0.9: `### DERIVE: \`id\``) only when no new-format matches are found
- `dedup-rewrites-both-antecedents-and-outlist` — When a duplicate is retracted via dedup, all justification references (both antecedent and outlist) across the network are rewritten to point at the kept node
- `derive-pipeline-is-error-accumulating` — The derive pipeline never raises exceptions to callers; `validate_proposals` returns skipped entries with reason strings, `apply_proposals` catches exceptions and returns error strings, and `apply_dedup_plan` collects errors into a list
- `dedup-auto-keeps-most-dependents` — In auto mode, `deduplicate` retains the cluster member with the most dependents and retracts all others

