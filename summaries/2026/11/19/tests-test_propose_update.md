---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_propose_update.py

## Purpose

`tests/test_propose_update.py` is the unit-test suite for the `propose-update` feature implemented in `reasons/propose_update.py`. Its job is to pin down the behavior of the model-assisted workflow that inspects existing beliefs, asks an LLM for proposed updates/retractions, parses the LLM response, and renders human-readable proposal markdown.

It covers four layers of the feature:

1. Formatting a belief’s context before sending it to the model.
2. Parsing robustly from free-form LLM output containing JSON.
3. Rendering individual proposals and the full proposals file.
4. Orchestrating model calls, batching, callbacks, and failure handling.

The suite imports directly from `reasons.propose_update`, not `reasons_lib.propose_update`, so it only protects the `reasons` package implementation even though the repository contains a parallel `reasons_lib` tree.

## Key Components

### Imported functions under test

- `format_belief_for_update(belief_id, nodes)`
  - Returns a markdown-ish context block for one belief.
  - Includes heading, text, source/source URL, type, dependents, stale reason, and justification details.
  - Returns `""` for a missing belief.

- `parse_update_proposals(response)`
  - Extracts proposal JSON from arbitrary LLM output, including fenced JSON and prose.
  - Returns a normalized list of proposal dictionaries.
  - Skips entries without `id`.
  - Defaults invalid enum-like fields instead of raising.

- `format_proposal_markdown(proposal, nodes=..., cascade=...)`
  - Renders one proposal as markdown.
  - Uses `UPDATE` or `RETRACT` header depending on `action`.
  - Shows current text from `nodes`, proposed text when applicable, failure mode, basis, evidence, comment, and suggested `reasons` command.
  - Renders cascade impact if provided.

- `format_proposals_file(proposals, nodes)`
  - Renders the complete proposals document.
  - Returns a placeholder when there are no proposals.

- `propose_updates(nodes, belief_ids=..., model=..., timeout=..., batch_size=..., on_batch=...)`
  - Main orchestration function.
  - Selects beliefs, batches them, calls `invoke_model`, parses proposals, accumulates results, and invokes an optional per-batch callback.
  - Swallows model errors and returns an empty result in the tested failure case.

### `sample_nodes` fixture

The fixture builds a small belief graph:

- `premise-a`: sourced premise, dependent on `derived-b`.
- `premise-stale`: sourced premise with `metadata.stale_reason = "content_changed"`.
- `derived-b`: derived belief with one justification whose antecedent is `premise-a` and label `"weather inference"`.
- `derived-c`: derived belief with one justification whose antecedent is `derived-b` and outlist entry `bad-weather-warning`.
- `bad-weather-warning`: an `OUT` belief with `derived-c` as a dependent.

This gives the tests cases for premises, derived beliefs, outlists, stale metadata, dependents, and multiple justifications.

### Test classes

- `TestFormatBeliefForUpdate`
  - Verifies premise formatting includes source fields and dependents.
  - Verifies derived formatting resolves antecedent text and labels.
  - Verifies outlist rendering as `"Unless (must be OUT):"`.
  - Verifies stale metadata rendering.
  - Verifies missing-node behavior returns `""`.
  - Verifies justification numbering when a belief has multiple justifications.

- `TestParseUpdateProposals`
  - Verifies valid JSON extraction from fenced blocks.
  - Verifies retract proposals preserve `proposed_text = None`.
  - Verifies multiple proposals preserve order.
  - Verifies malformed JSON returns `[]`.
  - Verifies invalid `basis` defaults to `"prior-knowledge"`.
  - Verifies invalid `failure_mode` defaults to `""`.
  - Verifies invalid `action` defaults to `"update"`.
  - Verifies prose around JSON is tolerated.
  - Verifies items without `id` are skipped.

- `TestFormatProposalMarkdown`
  - Verifies update proposal rendering includes current text, proposed text, failure mode, basis, evidence, and command.
  - Verifies retract proposal rendering omits proposed text and suggests `reasons retract ...`.
  - Verifies cascade rendering for retractions uses `"will go OUT"`.
  - Verifies cascade rendering for updates uses `"need re-review"`.
  - Verifies missing cascade says `"not computed"`.
  - Verifies empty cascade says `"none"`.

- `TestFormatProposalsFile`
  - Verifies empty input produces `"No updates proposed."`.
  - Verifies non-empty input produces `"# Proposed Updates"`, includes the proposal body, and uses `"---"` separators.

- `TestProposeUpdates`
  - Patches `reasons.propose_update.invoke_model` to avoid real model calls.
  - Verifies a basic update flow returns parsed proposals.
  - Verifies empty model response returns `[]`.
  - Verifies batching: 3 belief IDs with `batch_size=2` results in 2 model calls.
  - Verifies default behavior includes `IN` beliefs such as `premise-a` and `derived-b` in the prompt.
  - Verifies `RuntimeError` from the model is swallowed and returns `[]`.
  - Verifies `on_batch` callback receives per-batch results.
  - Verifies empty `belief_ids` short-circuits to `[]`.
  - Verifies all-nonexistent `belief_ids` short-circuits to `[]`.

## Patterns

- **Pytest class grouping**: tests are grouped by production function under test.
- **Fixture-based test graph**: `sample_nodes` is a reusable miniature belief store.
- **Mocking the LLM boundary**: `@patch("reasons.propose_update.invoke_model")` isolates model calls.
- **Behavioral string assertions**: formatting tests assert on specific substrings rather than whole rendered output.
- **Contract testing for tolerant parsing**: malformed JSON, invalid enums, missing IDs, and prose-wrapped JSON are explicitly tested.
- **Graceful degradation assertions**: model failure and invalid input are expected to produce empty results rather than exceptions.
- **Callback behavior testing**: `on_batch` is verified through a captured list of batch result counts.

## Dependencies

### This file depends on

- `json` for serializing proposal arrays in tests.
- `unittest.mock.patch` for replacing `invoke_model`.
- `pytest` for fixtures.
- `reasons.propose_update`:
  - `format_belief_for_update`
  - `format_proposal_markdown`
  - `format_proposals_file`
  - `parse_update_proposals`
  - `propose_updates`

### Runtime dependency indirectly exercised

- `reasons.propose_update.invoke_model` is the model boundary. Tests patch it in every test where a model call would otherwise occur.

### What depends on this file

- This file is collected by pytest.
- No production code imports it.
- It does not cover `reasons_lib/propose_update.py`; changes to that duplicate module would not be caught here.

## Flow

The general execution flow of the tested production behavior is:

1. **Build belief context**
   - `format_belief_for_update(id, nodes)` looks up the node.
   - If missing, returns `""`.
   - Otherwise renders text, source metadata, type, stale reason, dependents, and justifications.
   - Antecedent IDs are resolved to belief text when available.

2. **Select and batch beliefs**
   - `propose_updates` receives `nodes`.
   - If explicit `belief_ids` are given, it uses those.
   - If not, tests indicate the default targets `IN` beliefs and includes them in the prompt.
   - Empty or all-nonexistent IDs short-circuit before any model call.
   - Belief IDs are chunked according to `batch_size`.

3. **Call the model**
   - For each batch, the prompt includes formatted context blocks.
   - `invoke_model` is called once per batch.
   - Tests confirm 3 IDs with `batch_size=2` produce 2 calls.

4. **Parse model output**
   - `parse_update_proposals` extracts a JSON array even if surrounded by prose or fenced code.
   - It normalizes valid-looking proposal dictionaries.
   - Invalid enum-like fields are defaulted.
   - Items without `id` are skipped.

5. **Accumulate and notify**
   - Parsed proposals are accumulated across batches.
   - If `on_batch` is provided, it is called with batch results. The callback test confirms the callback sees `[1]` for a response containing one proposal.

6. **Render output**
   - Individual proposals are rendered by `format_proposal_markdown`.
   - The full document is rendered by `format_proposals_file`.
   - Cascade wording depends on the action: retraction says dependents `"will go OUT"`, update says dependents `"need re-review"`.

## Invariants

- `format_belief_for_update("missing-id", nodes)` returns `""`; it does not raise.
- `format_belief_for_update` includes `"Dependents"` only when the belief has dependents.
- Multiple justifications are numbered, e.g. `"Justification 1/2:"` and `"Justification 2/2:"`.
- `parse_update_proposals` is total for malformed input: malformed JSON returns `[]`.
- `parse_update_proposals` skips entries without an `id`.
- Invalid `basis` defaults to `"prior-knowledge"`.
- Invalid `failure_mode` defaults to `""`.
- Invalid `action` defaults to `"update"`.
- Proposal order from the LLM JSON is preserved.
- `format_proposal_markdown` does not render proposed text for retract proposals.
- Cascade rendering distinguishes:
  - missing cascade: `"not computed"`
  - zero affected: `"none"`
  - non-empty retract cascade: `"will go OUT"`
  - non-empty update cascade: `"need re-review"`
- `format_proposals_file([], nodes={})` returns `"No updates proposed."`.
- `propose_updates` with empty `belief_ids` returns `[]`.
- `propose_updates` with only nonexistent IDs returns `[]`.
- `propose_updates` batches model calls instead of always sending one giant prompt.

## Error Handling

- `parse_update_proposals` intentionally swallows malformed JSON and invalid field values. It returns either an empty list or normalized proposal dictionaries.
- `propose_updates` catches `RuntimeError` from `invoke_model` and returns `[]` in the tested single-batch case.
- `format_belief_for_update` handles missing belief IDs by returning an empty string.
- The tests do not assert logging or exception types beyond `RuntimeError`; a maintainer should inspect `reasons/propose_update.py` to see whether all exceptions are caught or only specific ones.
- The suite favors graceful degradation over strict failure: invalid LLM output and model errors are expected to produce empty/partial results rather than propagate exceptions.

## Topics to Explore

- [file] `reasons/propose_update.py` — Implementation under test; inspect prompt construction, JSON extraction, enum validation, and batching details.
- [function] `reasons/propose_update.py:propose_updates` — Verify how default belief selection handles `IN`/`OUT` values, how batches are built, and whether non-`RuntimeError` exceptions are caught.
- [file] `reasons_lib/propose_update.py` — Compare the parallel `reasons_lib` version because this test suite does not cover it.
- [general] `llm-response-parsing` — Explore how the production parser locates JSON arrays, strips fences, and normalizes proposal fields.
- [file] `tests/conftest.py` — Check for shared pytest fixtures or configuration that affects how this suite is collected and run.

## Beliefs

- `propose-update-tests-patch-invoke-model` — Model-calling tests replace `reasons.propose_update.invoke_model` with a mock to avoid real LLM invocation.
- `parse-update-proposals-returns-empty-list-on-malformed-json` — `parse_update_proposals("not json at all")` returns `[]` rather than raising.
- `invalid-proposal-enums-default` — Invalid `basis`, `failure_mode`, and `action` values normalize to `"prior-knowledge"`, `""`, and `"update"` respectively.
- `propose-updates-batches-belief-ids` — `propose_updates` calls `invoke_model` once per batch; three belief IDs with `batch_size=2` produce two calls.
- `propose-updates-swallows-model-errors` — A `RuntimeError` from `invoke_model` causes `propose_updates` to return `[]`.
