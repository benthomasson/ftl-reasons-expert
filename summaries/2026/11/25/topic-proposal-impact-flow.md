---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Trace how `what_if_retract`'s result is persisted in `proposals.impact_json` by `propose_retraction` and later consulted during `accept_proposal` drift checks.

## Trace

1. **Impact calculation**: `what_if_retract(...)` is defined at `reasons/api.py:512`. It produces the retraction-impact payload; the definition itself does not persist anything.

2. **Proposal persistence**: `propose_retraction(...)` starts at `reasons/api.py:5217`. Within the proposal-creation path, `impact_json` is referenced at `reasons/api.py:5262`, and the proposal-writing SQL column lists at `reasons/api.py:5347` and `reasons/api.py:5424` include `snapshot_json, impact_json, tags_json, created_at, updated_at`. That maps to the database column declared in `reasons/storage.py:110`:
   ```sql
   impact_json     TEXT DEFAULT '{}',
   ```
   So the retraction impact/result is carried into `proposals.impact_json` when the proposal row is written.

3. **Acceptance and drift checks**: `accept_proposal(...)` is defined at `reasons/api.py:5622`. It reloads the proposal with:
   - `SELECT ... snapshot_json, impact_json FROM proposals ...` at `reasons/api.py:5643`
   - `snapshot_json, impact_json = row` at `reasons/api.py:5659`

   Its docstring at `reasons/api.py:5631-5633` states that acceptance re-validates against the live network and marks the proposal stale on drift. The observed drift messages at `reasons/api.py:5719` and `reasons/api.py:5740` compare `snapshot['truth_value']`, showing that at least the stored snapshot is used in truth-value drift checks. The stored `impact_json` is loaded into the same accept path and is available for the retraction-specific drift/application checks.

4. **MCP entry point**: `reasons/mcp_server.py:484` exposes another `accept_proposal(proposal_id, voter)` entry point. Its docstring at `reasons/mcp_server.py:485-488` says it applies the change after drift re-validation; the observed implementation that reads `impact_json` is the API implementation at `reasons/api.py:5622`.

**Important gap**: The observations confirm the storage/load boundaries, but they do not show the exact line inside `propose_retraction` where `what_if_retract(...)` is called and serialized into `impact_json`, nor the exact line inside `accept_proposal` where parsed `impact_json` keys are consulted in a drift branch. To fully prove the end-to-end field-level trace, inspect the complete bodies of `reasons/api.py:5217` and `reasons/api.py:5622`.

## Topics to Explore

- [function] `reasons/api.py:propose_retraction` — Locate the exact `what_if_retract` call and the JSON serialization into the `impact_json` column.
- [function] `reasons/api.py:accept_proposal` — Inspect how the loaded `impact_json` is parsed and which drift branches consult it.
- [function] `reasons/api.py:what_if_retract` — Understand the shape of the impact payload that is persisted.
- [file] `reasons/storage.py` — Confirm the `proposals` schema and any migrations/defaults for `impact_json`.
- [general] `proposal-drift-tests` — Read the drift tests around `tests/test_api.py:1240-1263` and the CLI/MCP drift tests to see which fields are asserted stale.

## Beliefs

- `what-if-retract-is-defined-at-reasons-api-512` — `what_if_retract` is defined in `reasons/api.py` at line 512.
- `proposals-impact-json-is-text-default-empty-json` — The proposals table declares `impact_json TEXT DEFAULT '{}'` at `reasons/storage.py:110`.
- `propose-retraction-writes-impact-json-column` — `propose_retraction` references `impact_json` at `reasons/api.py:5262` and includes it in proposal-writing SQL at `reasons/api.py:5347` and `5424`.
- `accept-proposal-loads-impact-json` — `accept_proposal` selects `snapshot_json, impact_json` at `reasons/api.py:5643` and unpacks both at `reasons/api.py:5659`.
- `accept-proposal-drift-revalidation-is-documented` — The `accept_proposal` docstring at `reasons/api.py:5631-5633` states that acceptance re-validates against the live network and marks drift as stale.
