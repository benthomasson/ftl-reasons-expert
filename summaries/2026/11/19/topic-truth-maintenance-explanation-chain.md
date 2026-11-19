---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Understand the semantics of justification steps, antecedents, outlists, and root premises in this project.

## Semantics overview

This project implements a justification-based truth-maintenance network. Nodes have a truth value (`IN`/`OUT`) and can carry one or more `Justification` objects. The key idea is that a belief is `IN` when at least one of its justifications is valid; each justification describes the support and defeat conditions for that belief.

### `Justification` fields

`reasons/__init__.py:7` defines `Justification`. The relevant fields are documented at `reasons/__init__.py:11-21`:

- `antecedents` — the “inlist”: node IDs that must be `IN` for the justification to hold.
- `outlist` — node IDs that must be `OUT` for the justification to hold.
- `type` — `SL` is the main support-list type shown in the observations; `CP` also exists but its full truth rule is not shown in the provided extracts.
- `label` — human-readable annotation, used in tests like `tests/test_outlist.py`.

The core SL rule is stated directly in `reasons/__init__.py:11-12`:

> node is IN iff ALL antecedents (inlist) are IN AND ALL outlist nodes are OUT

That means an SL justification is a conjunction over both lists. However, a node can have multiple justifications, and those are alternatives: if one justification is valid, the node can remain `IN`. This is demonstrated by `TestOutlistWithMultipleJustifications.test_alternate_justification_survives`: when one justification is blocked by its outlist, a second justification can keep the node `IN`.

### Antecedents / inlist semantics

Antecedents are stored as node-ID strings. The API accepts comma-separated `sl` and `cp` arguments (`reasons/api.py:300-301`, `reasons/api.py:421-422`) and converts them into `Justification.antecedents` (`reasons/api.py:323-331`).

For SL justifications:

- All antecedents must be `IN` for that justification to hold.
- `any_mode` changes this shape: a multi-antecedent SL justification is expanded into one justification per antecedent (`reasons/api.py:306`, `reasons/api.py:324-326`). This makes the antecedents alternatives rather than a conjunction.

Namespace handling is applied to antecedents and outlists before persistence or evaluation:

- `reasons/api.py:367-368` resolves namespaces in `j.antecedents` and `j.outlist`.
- `reasons/api.py:454-455` resolves namespaces for the second API path.
- `reasons/api.py:350-360` also prepends a namespace-active antecedent when appropriate.

### Outlist semantics

The outlist is the non-monotonic part of the model. `reasons/__init__.py:15` describes it as:

> The outlist enables non-monotonic reasoning: “believe X unless Y is believed.”

Operationally:

- Every outlist node must be `OUT` for the justification to hold.
- If an outlist node is absent from the network, it is treated as `OUT`. This is verified by `TestOutlistBasic.test_outlist_node_not_in_network`.
- Multiple outlist nodes form a conjunction: all must be `OUT` (`TestOutlistCascading.test_multiple_outlist_nodes`).
- Asserting an outlist node invalidates the dependent belief and cascades through dependent nodes (`TestOutlistCascading.test_assert_outlist_node_retracts_dependent`, `test_outlist_cascade_through_chain`).
- Retracting the outlist node restores the dependent belief (`TestOutlistCascading.test_retract_outlist_node_restores_dependent`).
- Outlist data survives SQLite persistence and still propagates after load (`TestOutlistPersistence.test_round_trip`).

The API layer maps `unless` to `outlist` (`reasons/api.py:302`, `reasons/api.py:320`, `reasons/api.py:437`). An `unless`-only node is represented as an SL justification with no antecedents and the given outlist (`reasons/api.py:332-334`). That is how a default belief can hold unless some blocker is believed.

The defeat API also uses outlists rather than deleting the justification: at `reasons/api.py:1331-1394`, defeating a justification appends the defeater ID to `justification.outlist` if not already present. A similar path exists around `reasons/api.py:1526-1527`.

### Justification steps / explanation

The project exposes explanation records through `Network.explain`, as exercised in `tests/test_outlist.py`.

Relevant tests show that explanation steps include:

- `truth_value` — the node’s final status.
- `outlist` — the justification’s outlist.
- `violated_outlist` — the outlist nodes that are `IN`, and therefore block the justification.

See `TestOutlistExplain.test_explain_shows_outlist_on_valid` and `TestOutlistExplain.test_explain_shows_violated_outlist`. So an explanation step is not just a raw justification; it is an evaluated record showing which support/defeat conditions succeeded or failed.

The evaluator’s outlist check is visible around `reasons/api.py:1590-1592`:

```python
outlist_ok = all(
    ...
    for o in j.outlist
)
```

That reinforces that outlist validity is “all entries must be OUT.”

### Root premises

“Root premises” are a reporting concept rather than a field on `Justification`. The evidence-report code describes tracing a belief back to its root premises and then showing source evidence for each premise:

- `reasons/report_belief.py:9` — “trace a belief back to its root premises and the source evidence supporting each premise.”
- `reasons/report_belief.py:17` — “For each root premise, present the source evidence that supports it.”
- `reasons/api.py:3740` — same idea in the API docstring.
- `reasons/mcp_server.py:239` — “Find all root premises a belief ultimately depends on.”

So semantically, a root premise is a terminal/foundational node in the dependency trace: the belief’s supporting premises that are reported at the leaves of the justification graph. The provided observations do not show the exact algorithm that classifies a node as a root premise, but the reporting intent is clear.

### What is not fully specified in the observations

Two areas are partially visible but not fully defined:

- `CP` justifications are created by the API (`reasons/api.py:331`, `reasons/api.py:466`), but the extracted code does not show the CP truth rule.
- The exact root-premise-selection algorithm is not shown; only the reporting descriptions and tests are visible.

## Topics to Explore

- [file] `reasons/__init__.py` — Defines `Justification` and the top-level SL/outlist semantics.
- [function] `reasons/api.py:add_node` — Builds SL/CP justifications from `sl`, `cp`, `unless`, and `any_mode`.
- [file] `tests/test_outlist.py` — Exercises outlist cascades, multiple justifications, persistence, and explanation steps.
- [file] `reasons/report_belief.py` — Implements root-premise tracing and evidence reporting.
- [general] `cp-justification-semantics` — The observations show CP exists, but the full truth rule still needs to be traced through the network evaluator.

## Beliefs

- `sl-justification-requires-all-antecedents-in` — An SL justification is valid only when every ID in its `antecedents` list is `IN`.
- `sl-justification-requires-all-outlist-nodes-out` — An SL justification is valid only when every ID in its `outlist` is `OUT`; missing outlist nodes count as `OUT`.
- `node-in-requires-any-valid-justification` — A node with multiple justifications is `IN` if at least one justification is valid.
- `any-mode-expands-sl-to-or` — `any_mode` converts a multi-antecedent SL into one justification per antecedent, making the antecedents alternatives.
- `defeat-uses-outlist` — The defeat API appends the defeater ID to a justification’s `outlist` rather than deleting or rewriting the justification.
