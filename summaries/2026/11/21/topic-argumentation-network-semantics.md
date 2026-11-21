---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: IN/OUT truth, `SL` justifications, `unless` outlists, nogoods, entailment, and backtracking are the domain concepts all these tests encode; a formal model (e.g., defeasible logic) will make the diamond/diamond-restoration and “retracted pin skipped” tests obvious.

# Mental model: a defeasible truth-maintenance network

Treat this codebase as a defeasible truth-maintenance system. Each node has a truth value, `IN` or `OUT`. A base premise is an assumption that is `IN` unless retracted/challenged. An `SL` justification is a default rule:

```text
conclusion is IN if:
  all antecedents are IN
  AND all outlist nodes are OUT
```

A node is `IN` if at least one of its justifications fires. A nogood is an integrity constraint saying “these nodes cannot all be `IN`.” Backtracking resolves a violated nogood by tracing back through justifications to accountable premises and retracting one of them.

This model explains the tests across `tests/test_outlist.py`, `tests/test_backtracking.py`, `tests/test_nogood_id.py`, and `tests/test_pin_sources.py`. The observation JSON exposes line windows (`start_line` / `lines_returned`) rather than stable per-test line numbers, so I’ll cite the file and symbol inside those returned windows.

## IN/OUT truth and entailment

`IN` means “currently believed / entailed in the current context.” `OUT` means “not currently believed.”

Base premises are normally added `IN`, then can be retracted to `OUT`:

- `tests/test_outlist.py`: `TestOutlistBasic.test_outlist_valid_when_out` adds `y`, retracts `y`, then adds `x` with `Justification(type="SL", antecedents=[], outlist=["y"])`; `x` is `IN`.
- `tests/test_outlist.py`: `TestOutlistCascading.test_assert_outlist_node_retracts_dependent` asserts `y` back `IN` and expects dependent `x` to become `OUT`.
- `tests/test_outlist.py`: `TestOutlistCascading.test_retract_outlist_node_restores_dependent` retracts `y` and restores `x` to `IN`.

Derived entailment is existential over justifications: a node is `IN` if **any** justification is valid. This is why `TestOutlistWithMultipleJustifications.test_alternate_justification_survives_outlist` works: `x` has two justifications, one defeated by `y` becoming `IN`, but the other `SL(["a"])` keeps `x` `IN` until `a` is retracted.

Entailment is non-monotonic: adding a belief can retract another belief. Asserting an outlist node defeats the default justification. Retracting it restores the default. The cascade tests show entailment propagating through dependents: in `TestOutlistCascading.test_outlist_cascade_through_chain`, asserting `y` makes `x` `OUT`, which makes dependent `z` `OUT`.

## `SL` justifications and `unless` outlists

`Justification(type="SL", ...)` encodes a support-list/default rule. The relevant fields are:

- `antecedents`: nodes that must be `IN`.
- `outlist`: nodes that must be `OUT` — the “unless” part.

The validity condition is conjunctive over both lists:

```text
valid(SL(A, O), node n) iff
  every a in A is IN
  and every o in O is OUT
```

Key tests:

- `TestOutlistBasic.test_outlist_valid_when_out`: `x` holds unless `y`; `y` is `OUT`, so `x` is `IN`.
- `TestOutlistBasic.test_outlist_invalid_when_in`: `y` is `IN`, so `x` is `OUT`.
- `TestOutlistBasic.test_outlist_with_inlist`: `x` requires `a` `IN` and `y` `OUT`.
- `TestOutlistBasic.test_outlist_with_inlist_both_fail`: if `a` is `OUT`, `x` is `OUT` regardless of `y`.
- `TestOutlistBasic.test_outlist_node_not_in_network`: an absent outlist node is treated as `OUT`; absence means “not believed.”
- `TestOutlistCascading.test_multiple_outlist_nodes`: `outlist=["y", "z"]` is conjunctive — both must be `OUT`; asserting just `y` retracts `x`.

`TestOutlistExplain` also confirms the explanation layer exposes `outlist` and `violated_outlist`, so the semantics are observable in traces.

`reasons/derive.py`’s `DERIVE_PROMPT` describes the same intended semantics: an outlist-gated conclusion is `OUT` while the outlist node is `IN`, and flips `IN` when it goes `OUT`. `parse_proposals` also parses `### GATE` proposals with an `- Unless:` field.

## Nogoods and backtracking

A nogood is a constraint over nodes: those nodes cannot all be `IN` together. `tests/test_nogood_id.py` defines nogood IDs from sorted node IDs:

- `TestContentBasedIds.test_id_from_sorted_nodes`: `add_nogood(["b", "a"])` produces `nogood-a-b`.
- `TestContentBasedIds.test_duplicate_gets_suffix`: duplicate nogoods get `nogood-a-b-2`.
- `TestContentBasedIds.test_three_node_nogood`: `["z", "x", "y"]` becomes `nogood-x-y-z`.

When a nogood is violated, `add_nogood` does dependency-directed backtracking rather than retracting an arbitrary node:

- `tests/test_backtracking.py`: `TestTraceAssumptions.*` walks antecedents backward to root premises. `test_premise_traces_to_itself`, `test_derived_traces_to_premise`, `test_chain_traces_to_root`, and `test_diamond_traces_to_root` all show `trace_assumptions` returning the underlying assumptions, deduplicated.
- `TestFindCulprits.*` maps contradiction members back to responsible premises. `test_derived_nodes_trace_to_premise` shows `c` derived from `a`; a nogood over `["b", "c"]` identifies `a` and `b` as culprits.
- `test_sorted_by_dependents` shows culprit ordering prefers premises with fewer dependents.
- `test_would_resolve_field` shows culprits report which nogood members they would resolve.

Then `TestBacktrackingInNogood` verifies the result:

- `test_retracts_premise_not_derived`: for `A -> C`, premise `B`, and `nogood(B, C)`, the system retracts a premise (`b` in this case), not derived `c`; `c` stays `IN` because `a` still supports it.
- `test_cascade_from_retracted_premise`: retracting the chosen premise can cascade derived nodes `OUT`, but the test selects `d` with zero dependents over `a`.
- `test_backtracking_logs_culprit`: log contains `action == "backtrack"` with a `culprit`.
- `test_shared_premise_resolves_both`: `A -> B`, `A -> C`, `nogood(B, C)` retracts shared premise `A`, resolving both.

Formally, backtracking is choosing an assumption whose removal falsifies at least one nogood member while minimizing disruption, typically by dependent count.

## Diamond and “retracted pin skipped”

The provided `tests/test_backtracking.py` snippet contains `TestTraceAssumptions.test_diamond_traces_to_root`:

```text
A -> B
A -> C
B + C -> D
```

`trace_assumptions("d")` returns `["a"]`. In the defeasible model, `D` is entailed only when both `B` and `C` are `IN`, and both branches trace to the same root assumption `A`. If `A` is retracted, `B`, `C`, and `D` go `OUT`; if `A` is restored, both branches restore and `D` can restore. That is the diamond/restoration intuition.

However, the explicit `search_diamond_restoration_retracted_pin_terms` result in the observations is empty. The exact “diamond-restoration” and “retracted pin skipped” tests are not present in the observed snippets. The nearest observed pinning test is:

- `tests/test_pin_sources.py`: `TestPinSources.test_skips_out_nodes` manually sets a node’s `truth_value = "OUT"` and expects `pin_sources` to return no results.

That matches the intended rule: pinning operates on currently `IN` beliefs with sources. A retracted node is `OUT`, so it is skipped, along with already-pinned nodes and nodes without sources, as shown by `test_skips_already_pinned`, `test_skips_nodes_without_source`, and `test_skips_nonexistent_source`.

So: the formal model makes the general behavior obvious, but the observations do not include the exact tests named “diamond-restoration” or “retracted pin skipped.” We only have adjacent evidence.

## Topics to Explore

- [file] `tests/test_backtracking.py` — Full trace_assumptions, find_culprits, and add_nogood tests; inspect exact diamond and culprit-ranking expectations.
- [function] `reasons/network.py:add_nogood` — Core nogood resolution and backtracking logic; verify how culprits are selected and logged.
- [function] `reasons/network.py:trace_assumptions` — Backward traversal from derived nodes to root premises; central to diamond handling.
- [file] `tests/test_pin_sources.py` — Confirms pinning skips `OUT`, already-pinned, sourceless, and nonexistent-source nodes.
- [general] `defeasible-logic-model` — Model `SL` rules as defaults and nogoods as integrity constraints to reason about restoration and retraction.

## Beliefs

- `sl-justification-validity-requires-antecedents-in-and-outlist-out` — An `SL` justification fires only when all antecedents are `IN` and all outlist nodes are `OUT`.
- `node-in-truth-is-existential-over-justifications` — A node is `IN` if at least one justification fires, or if it is an unretracted premise; otherwise it is `OUT`.
- `outlist-changes-cascade-to-dependents` — Asserting an outlist node retracts dependent beliefs, and retracting it restores them.
- `nogood-resolution-backtracks-to-premise` — `add_nogood` traces contradiction members to root premises and retracts a premise, preferring minimal disruption such as fewer dependents.
- `pin-sources-skips-out-nodes` — `pin_sources` ignores nodes whose `truth_value` is `OUT`, as tested in `tests/test_pin_sources.py:TestPinSources.test_skips_out_nodes`.
