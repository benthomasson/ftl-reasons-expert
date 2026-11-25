---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: _build_justifications in reasons/import_agent.py

## Purpose

`_build_justifications` is a translation/adapter function. It converts the normalized, source-relative justification records on an imported claim into domain `Justification` objects suitable for the local `Network`.

It exists to centralize three import rules:

1. Namespace every imported antecedent/outlist belief ID with an agent prefix such as `"aap-expert:"`.
2. Inject the agent’s `inactive_id` — e.g. `"aap-expert:inactive"` — into every generated justification’s `outlist` so the agent-level kill switch can retract all imported beliefs.
3. Provide a deterministic fallback `Justification` when a claim has no raw justifications, so imported nodes still get a consistent, kill-switch-aware justification list.

It does **not** create nodes, set truth values, mark nodes as retracted, or validate that referenced nodes exist. Those responsibilities belong to its callers.

## Contract

### Preconditions

The function assumes:

- `claim` is a normalized claim dict, i.e. from `_normalize_markdown` or `_normalize_json`.
- `claim["raw_justifications"]` exists and is iterable.
- Each raw justification has:
  - `type`: usually `"SL"`
  - `antecedents`: iterable of source-relative IDs
  - `outlist`: iterable of source-relative IDs
  - `label`: string or `None`
- `prefix` is a namespace string, expected to include the delimiter, e.g. `"aap-expert:"`.
- `inactive_id` is a fully-qualified local node ID, e.g. `"aap-expert:inactive"`.
- `agent_name` is the agent name used for default labels.
- Raw IDs are unprefixed. The function blindly prepends `prefix`.

None of these assumptions are enforced by type annotations or runtime validation.

### Postconditions

- Always returns a non-empty `list[Justification]`.
- For each raw justification, returns exactly one `Justification` with:
  - `antecedents`: each raw antecedent prefixed with `prefix`
  - `outlist`: `[inactive_id]` followed by each raw outlist item prefixed with `prefix`
  - `type`: copied unchanged from the raw justification
  - `label`: raw label if truthy, otherwise `f"imported from agent: {agent_name}"`
- If `claim["raw_justifications"]` is empty, returns one fallback `Justification`:
  - `type="SL"`
  - `antecedents=[]`
  - `outlist=[inactive_id]`
  - default imported-agent label

### Invariants

- `inactive_id` is always the first element of every returned justification’s `outlist`.
- The function never inserts the agent’s `active_id` into `antecedents`. This is deliberate: the module uses `inactive_id` in `outlist` as the kill switch, not `active_id` as an always-valid antecedent.
- No deduplication is performed. If a raw outlist already maps to `inactive_id`, the returned outlist can contain `inactive_id` twice.
- No filtering against existing network nodes is performed. The caller/normalizer is responsible for filtering raw IDs when needed.

## Parameters

- `claim`: Normalized claim dict. Must contain `"raw_justifications"`. The function does not inspect `"is_out"` or any other claim fields.

- `prefix`: String namespace prefix, normally `f"{agent_name}:"`. It is concatenated directly with raw IDs. If it lacks a trailing colon, IDs become malformed. If empty, no namespacing occurs.

- `inactive_id`: Fully-qualified ID for the agent’s relay/kill-switch node. The function does not prefix it again. It expects the node to exist or be created by the caller.

- `agent_name`: Used only to build default labels. It is not used to construct IDs.

## Return Value

Returns a non-empty list of `Justification` objects.

The list length is:

- `len(claim["raw_justifications"])` when at least one raw justification exists.
- `1` when `raw_justifications` is empty, via the fallback justification.

The caller receives fresh `Justification` objects and fresh `antecedents`/`outlist` lists. The caller must decide whether the returned justifications should be attached to a new node, compared against an existing node, or replaced via sync logic.

Critically, the returned justifications do **not** encode whether the imported claim is `IN`, `OUT`, or retracted. A claim originally marked `OUT` with no raw justifications still gets the fallback `SL` justification here; the caller later applies retraction or `_retracted` metadata.

## Algorithm

1. Initialize an empty list `justs`.
2. For each raw justification `rj` in `claim["raw_justifications"]`:
   - Prefix every antecedent ID:
     ```python
     antecedents = [f"{prefix}{a}" for a in rj["antecedents"]]
     ```
   - Build the outlist starting with `inactive_id`, then append prefixed raw outlist IDs:
     ```python
     outlist = [inactive_id] + [f"{prefix}{o}" for o in rj["outlist"]]
     ```
   - Choose the label:
     ```python
     label = rj["label"] or f"imported from agent: {agent_name}"
     ```
     This means `None`, `""`, and other falsy labels are replaced by the default label.
   - Construct a `Justification` with the copied type and the transformed antecedent/outlist lists.
3. If no justifications were produced, create one fallback:
   ```python
   Justification(
       type="SL",
       antecedents=[],
       outlist=[inactive_id],
       label=f"imported from agent: {agent_name}",
   )
   ```
4. Return `justs`.

## Side Effects

None directly.

The function only allocates new lists and `Justification` objects. It does not mutate `claim`, the raw justification dicts, the network, or any node metadata. It does not perform I/O.

Any network state changes happen in callers such as `_import_claims` and `_sync_claims`, which add nodes, replace justifications, call `network.retract`, or rebuild dependents.

## Error Handling

No exceptions are caught or converted.

Potential runtime errors include:

- `KeyError` if `claim["raw_justifications"]` or keys such as `rj["antecedents"]`, `rj["outlist"]`, `rj["type"]`, or `rj["label"]` are missing.
- `TypeError` if `raw_justifications` is `None` or if an antecedent/outlist field is not iterable.
- Constructor errors from `Justification` if it validates its arguments and receives an invalid type or label.
- No validation that `prefix`, `inactive_id`, or `agent_name` are strings. Because f-strings are used, many non-string values are silently stringified when building IDs.
- No validation that referenced prefixed nodes exist.

## Usage Patterns

This helper is called from both import and sync paths:

- In `_import_claims`, it is called for every new claim before `network.add_node(...)`.
- In `_sync_claims`, it is called for:
  - new remote beliefs
  - existing `OUT` beliefs with no raw justifications
  - existing `OUT` beliefs with preserved justifications
  - existing `IN` beliefs
- In sync, the result is compared with existing `node.justifications` via `_justifications_match`; if changed, `_update_node_justifications` replaces the list and rebuilds dependents.

Caller obligations:

- Ensure the agent’s `inactive_id` node exists or will be created. Both main callers call `_ensure_agent_nodes` first.
- Pass normalized claims whose raw IDs are source-relative, not already prefixed.
- Handle `OUT`/retraction semantics separately. The function does not know about `claim["is_out"]`.
- Call `_fixup_dependents` or `_rebuild_dependents` after adding/replacing justifications when outlist nodes may be newly referenced.

## Dependencies

- `Justification` class imported at module level from the package:
  ```python
  from . import Justification, Nogood
  ```
- Normalized claim structure produced by:
  - `_normalize_markdown`
  - `_normalize_json`
- Agent infrastructure created by:
  - `_ensure_agent_nodes`
- Indirect network interaction through callers:
  - `Network.add_node`
  - `_update_node_justifications`
  - `_fixup_dependents`
  - `network.recompute_all`
  - `network.retract`

## Topics to Explore

- [file] `reasons/network.py` — Understand how `Justification` antecedents and outlists are evaluated during propagation, and what `retract` and `recompute_all` do.
- [function] `reasons/import_agent.py:_ensure_agent_nodes` — Shows how `agent:active` and `agent:inactive` are created and why `inactive_id` is the kill switch.
- [function] `reasons/import_agent.py:_normalize_json` — Explains the raw justification shape and which IDs are filtered before `_build_justifications`.
- [function] `reasons/import_agent.py:_sync_claims` — Demonstrates how returned justifications are compared and replaced during sync.
- [file] `tests/test_import_agent.py` — Tests the import/sync behavior around namespacing, outlists, and agent retraction.

## Beliefs

- `build-justifications-requires-normalized-unprefixed-raw-ids` — The function assumes raw antecedent and outlist IDs are source-local and blindly prepends `prefix`.
- `inactive-id-is-injected-into-all-outlists` — Every returned justification has `inactive_id` as the first element of its outlist, including the fallback.
- `empty-raw-justifications-produces-fallback-sl-justification` — If `claim["raw_justifications"]` is empty, the function returns one `SL` justification with no antecedents and outlist `[inactive_id]`.
- `build-justifications-does-not-mutate-network-state` — It only constructs `Justification` objects; truth values, `_retracted`, and retraction are handled by callers.
- `build-justifications-uses-falsy-label-fallback` — `label = rj["label"] or ...` replaces `None` and empty-string labels with the imported-agent default, while truthy labels are preserved.
