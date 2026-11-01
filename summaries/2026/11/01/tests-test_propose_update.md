---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_propose_update.py

# `tests/test_propose_update.py`

## 1. Purpose

This file is the test suite for the `reasons.propose_update` module — the subsystem that sends a set of beliefs to an LLM and asks it to **propose updates or retractions** (with structured justification). The propose-update workflow is a "review pass" over a belief graph: it formats beliefs into a prompt, invokes a model, parses the structured JSON response into validated proposal objects, and renders those proposals into human-readable Markdown (optionally with cascade-impact analysis).

This test file owns verifying:
- The **formatting** layer (belief → prompt markdown)
- The **parsing/validation** layer (raw LLM text → typed proposal dicts)
- The **rendering** layer (proposal dict → reviewable Markdown document)
- The **orchestration** layer (`propose_updates` batching, callbacks, error swallowing)

## 2. Key Components

### Fixture: `sample_nodes`

A 5-node belief graph used across most tests. It encodes a small dependency chain:

```
premise-a ──► derived-b ──► derived-c ◄── bad-weather-warning (OUT, in outlist)
premise-stale (standalone, flagged stale)
```

Each node is a dict with keys: `text`, `truth_value` (`"IN"`/`"OUT"`), `source`, `source_url`, `justifications` (list of `{antecedents, outlist, label}`), `dependents` (list of IDs), and `metadata`.

### `TestFormatBeliefForUpdate` (7 tests)

Tests `format_belief_for_update(node_id, nodes) -> str`. Contract:
- Returns a Markdown fragment starting with `### <id>`.
- Includes `Text:`, `Source:`, `Source URL:` fields for premises.
- For derived beliefs, expands `justifications` with `Antecedents:` (resolving each antecedent ID to its text) and `Unless (must be OUT):` for outlist entries.
- Shows `Dependents (N):` only when non-empty.
- Surfaces `metadata.stale_reason` if present.
- Returns `""` for an unknown node ID.
- Numbered justifications (`Justification 1/2:`) when multiple exist.

### `TestParseUpdateProposals` (10 tests)

Tests `parse_update_proposals(raw_response: str) -> list[dict]`. Contract:
- Accepts JSON either bare, inside a fenced code block, or surrounded by prose.
- Returns a list of validated dicts with keys: `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`.
- **Validation defaults** (the critical invariants):
  - `action` must be `"update"` or `"retract"`; invalid → defaults to `"update"`.
  - `basis` must be in a known set; invalid → defaults to `"prior-knowledge"`.
  - `failure_mode`: if not in a known set → defaults to `""`.
  - Items lacking an `id` key are **silently dropped**.
- Malformed JSON or non-list input → returns `[]` (never raises).

### `TestFormatProposalMarkdown` (6 tests)

Tests `format_proposal_markdown(proposal, nodes=..., cascade=None) -> str`. Contract:
- UPDATE proposals render `## UPDATE: <id>`, show both current and proposed text, and emit a CLI command hint (`reasons update <id>`).
- RETRACT proposals render `## RETRACT: <id>`, omit proposed text, and hint `reasons retract <id>`.
- Cascade rendering differs by action:
  - RETRACT + cascade → "will go OUT" language.
  - UPDATE + cascade → "need re-review" language (dependents aren't retracted, just invalidated).
  - No cascade → `"**Cascade impact:** not computed"`.
  - Empty cascade (`total_affected == 0`) → `"**Cascade impact:** none"`.

### `TestFormatProposalsFile` (2 tests)

Tests `format_proposals_file(proposals, nodes=...) -> str`. Contract:
- Empty list → `"No updates proposed."`
- Non-empty → `# Proposed Updates` header, each proposal separated by `---`.

### `TestProposeUpdates` (7 tests)

Tests the top-level orchestrator `propose_updates(nodes, belief_ids=None, model=..., timeout=..., batch_size=..., on_batch=None) -> list[dict]`. Uses `@patch("reasons.propose_update.invoke_model")` to stub the LLM call.

Key behaviors verified:
- **Batching**: 3 belief IDs with `batch_size=2` → exactly 2 `invoke_model` calls.
- **Default selection**: when `belief_ids` is `None`, all `"IN"`-valued nodes are included in the prompt.
- **Error swallowing**: if `invoke_model` raises `RuntimeError`, the function returns `[]` rather than propagating.
- **Callback**: `on_batch` is invoked once per batch with the parsed results list.
- **Guard clauses**: empty `belief_ids` or all-nonexistent IDs → `[]` without calling the model.

## 3. Patterns

| Pattern | Where |
|---|---|
| **Shared fixture via pytest** | `sample_nodes` is a function-scoped fixture injected into every test that needs the graph. Tests that need a *different* graph (e.g. `test_multiple_justifications`) build a local dict instead. |
| **Mock at the seam** | `@patch("reasons.propose_update.invoke_model")` patches the *imported symbol* in the module under test, not the source module. This is the standard "patch where it's looked up" rule. |
| **Class-per-function-under-test** | Each test class maps 1-to-1 with a function from `reasons.propose_update`. This keeps the mental model flat: class name tells you which contract you're checking. |
| **Default-validation table** | `TestParseUpdateProposals` has three near-identical tests (`test_invalid_basis_defaults`, `test_invalid_failure_mode_defaults`, `test_invalid_action_defaults`) that each feed one bad enum value and assert the fallback. This is a lightweight parameterization without `@pytest.mark.parametrize`. |
| **Assertion on rendered text** | Formatting tests assert on exact substrings (`"### premise-a"`, `"Dependents (1):"`). This couples the tests to the Markdown template — a deliberate choice so that template refactors break tests loudly. |

## 4. Dependencies

**Imports (this file →):**
- `reasons.propose_update` — the module under test. Specifically: `format_belief_for_update`, `format_proposal_markdown`, `format_proposals_file`, `parse_update_proposals`, `propose_updates`.
- `reasons.propose_update.invoke_model` — patched at runtime (not imported directly by the test).
- `pytest` — fixtures, test discovery.
- `unittest.mock.patch` — LLM call stubbing.
- `json` — serializing mock return values.

**Depended upon by (→ this file):**
- Nothing imports this test file. It's a leaf in the dependency graph, consumed only by the test runner.

## 5. Flow

The end-to-end flow that `TestProposeUpdates` exercises:

```
propose_updates(nodes, belief_ids, ...)
│
├─ 1. Resolve belief_ids (default: all IN nodes)
│     Filter out IDs not present in `nodes`.
│     If empty → return [].
│
├─ 2. Batch the IDs (default batch_size from module constant).
│
├─ 3. For each batch:
│     a. format_belief_for_update(id, nodes)  →  Markdown fragment per belief
│     b. Concatenate fragments into a single prompt string
│     c. invoke_model(prompt, model=..., timeout=...)  →  raw LLM text
│     d. parse_update_proposals(raw_text)  →  list[dict]  (validated)
│     e. on_batch(parsed)  if callback provided
│
└─ 4. Accumulate and return all parsed proposals.
```

The rendering path (used by CLI / export) is separate:

```
format_proposals_file(proposals, nodes)
  └─ for each proposal:
       format_proposal_markdown(proposal, nodes, cascade)
         └─ looks up current text from nodes[proposal["id"]]
```

## 6. Invariants

These are the rules the tests **enforce** and that a maintainer must not break:

1. **`parse_update_proposals` never raises.** Any malformed input (non-JSON, non-list, missing keys) results in `[]` or a filtered list. The function is a total function over `str`.
2. **Enum validation with safe defaults.** `action` falls back to `"update"`, `basis` to `"prior-knowledge"`, `failure_mode` to `""`. This means a proposal is always actionable even if the LLM hallucinates an enum value.
3. **Items without `id` are dropped, not defaulted.** There is no fallback ID — a proposal must identify *which* belief it targets.
4. **`format_belief_for_update` returns `""` for unknown IDs** rather than raising `KeyError`. Callers can concatenate results without try/except.
5. **Cascade language is action-dependent.** RETRACT → "will go OUT"; UPDATE → "need re-review". This is a semantic invariant: retracting a premise *destroys* dependents, while updating it only *invalidates* their justification.
6. **`propose_updates` with empty/invalid `belief_ids` short-circuits to `[]`** without invoking the model.

## 7. Error Handling

| Scenario | Behavior | Test |
|---|---|---|
| LLM returns non-JSON / garbage | `parse_update_proposals` returns `[]`; no exception | `test_malformed_json` |
| LLM raises `RuntimeError` | `propose_updates` catches it, returns `[]` | `test_handles_llm_error` |
| LLM returns valid JSON but invalid enum values | Fields are silently replaced with defaults | `test_invalid_*_defaults` (×3) |
| LLM returns proposals for non-existent belief IDs | They are included in parsed results (validation of ID existence happens upstream in `propose_updates` which filters the *input*, not the *output*) | `test_nonexistent_belief_ids` (tests the input side) |
| `format_belief_for_update` called with missing node | Returns `""` — no `KeyError` | `test_missing_node` |
| Cascade data is `None` | Renders `"not computed"` | `test_no_cascade` |
| Cascade data is empty (zero affected) | Renders `"none"` | `test_empty_cascade` |

The overall error-handling philosophy is **graceful degradation at every boundary**: the LLM is treated as an untrusted text source, the parser is a total function, and the orchestrator swallows transport errors. No test expects an exception to propagate out of any of the five public functions.

## Topics to Explore

- [file] `reasons/propose_update.py` — The module under test. Read the actual implementations of `parse_update_proposals`, `propose_updates`, and the allowed `basis`/`failure_mode`/`action` enums to understand the validation tables the tests assume.
- [file] `reasons/check_stale.py` — The `premise-stale` fixture node has `metadata.stale_reason: "content_changed"`. Understanding how staleness is detected and tagged tells you where the `stale` failure mode originates in the propose-update pipeline.
- [function] `reasons/propose_update.py:propose_updates` — The orchestration function. Examine how it builds the prompt (system message, formatting instructions), how `batch_size` is chunked, and how `on_batch` interacts with the return value.
- [file] `reasons/review.py` — The "re-review" language in UPDATE cascases implies that downstream dependents are re-evaluated by a review pass. Understanding that module clarifies the full lifecycle after a proposal is accepted.
- [general] `cascade-impact-model` — The `cascade` dict shape (`retracted`, `restored`, `total_affected`, per-node `depth` and `dependents`) appears in `format_proposal_markdown` but is produced elsewhere. Tracing its producer reveals the graph-traversal logic that computes transitive impact.

## Beliefs

- `parse-update-proposals-is-total-over-str` — `parse_update_proposals` accepts any string and always returns a list of dicts; it never raises, and malformed input yields `[]`.
- `invalid-enum-fields-default-to-safe-values` — An invalid `action` becomes `"update"`, an invalid `basis` becomes `"prior-knowledge"`, and an invalid `failure_mode` becomes `""`; proposals remain actionable regardless of LLM hallucination.
- `retract-cascade-uses-destructive-language-update-uses-revalidation-language` — In rendered Markdown, RETRACT cascades say dependents "will go OUT" while UPDATE cascades say dependents "need re-review", encoding the semantic difference between destroying and invalidating.
- `propose-updates-short-circuits-on-empty-or-missing-ids` — `propose_updates` returns `[]` immediately (without calling `invoke_model`) when `belief_ids` is empty or contains only IDs absent from the nodes dict.
- `test-mocks-invoke-model-at-module-import-site` — The LLM call is stubbed via `@patch("reasons.propose_update.invoke_model")`, meaning `invoke_model` must be a module-level name in `reasons/propose_update.py` (not a method or local import) for the patch to intercept it.
