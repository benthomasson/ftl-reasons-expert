---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The observations show CP exists, but the full truth rule still needs to be traced through the network evaluator.

## Explanation

CP exists in the codebase as a **justification type**, but the observations do not expose the full semantic predicate that decides whether a CP node is `IN` or `OUT`.

### What the observations prove

- CP is explicitly modeled as a first-class justification type:
  - `reasons/__init__.py:13` documents CP as “node is IN iff assumptions are consistent.”
  - `reasons/__init__.py:19` declares `type: str  # "SL" or "CP"`.
  - `reasons/pg.py:45` enforces `CHECK (type IN ('SL', 'CP'))`.

- CP input is parsed into a `Justification`:
  - `reasons/api.py:281`, `301`, `316`, `329-331` accept `cp="A,B"`, split it into antecedents, and create `Justification(type="CP", antecedents=..., outlist=..., label=...)`.
  - The same pattern appears again around `reasons/api.py:409-449`.
  - `reasons/cli.py:3077` and `reasons/cli.py:3095` expose `--cp`.
  - `reasons/pg.py:3668-3681` parses stored CP justifications.

- The network layer knows about CP:
  - `reasons/network.py:4` says nodes have `SL or CP` justifications.
  - `reasons/network.py:982` has `if j.type in ("SL", "CP"):`, showing CP is handled in a shared network branch.
  - In the observed `Network.add_node` excerpt, any node with justifications gets `node.truth_value = self._compute_truth(node)`. That means CP justifications feed into the same network evaluator as SL justifications.

### Why the full truth rule is still missing

The missing piece is the body of the network evaluator, especially `Network._compute_truth` and the propagation path.

From the observed `reasons/network.py` excerpt:

- `Network.add_node` delegates initial truth computation to `self._compute_truth(node)` when justifications exist.
- `Network.retract` and `Network.assert_node` update the target node and call `self._propagate(...)` to cascade truth changes.
- The actual `_compute_truth` implementation is not included in the returned lines `1-500`.
- The only CP-specific network evidence is `reasons/network.py:982`, which proves CP enters a shared branch but does not show the exact condition for CP truth.
- The search for `full truth`, `truth rule`, or evaluator definitions returned no useful matches, so “full truth rule” is not a literal named function in the supplied observations.

The docstring says CP is `IN` iff assumptions are consistent, but “consistent” is not defined by the observations. It is not enough to know whether CP means:

- all CP antecedents must be `IN`,
- no active nogood may involve the CP assumptions,
- outlist nodes must be `OUT`,
- or some combination handled inside `_compute_truth` / `_propagate`.

The provided tests mostly exercise SL behavior: all antecedents required, retraction cascade, restoration, alternate justifications. The first 400 lines of `tests/test_network.py` do not show CP-specific truth tests, so the expected CP semantics are not pinned down by that excerpt.

Bottom line: CP exists throughout parsing, persistence, and network registration. The full CP truth rule must be traced by reading `reasons/network.py` around `Network._compute_truth`, `Network._propagate`, and `reasons/network.py:982`.

## Topics to Explore

- [function] `reasons/network.py:Network._compute_truth` — The missing evaluator body that likely defines CP `IN`/`OUT` semantics.
- [function] `reasons/network.py:Network._propagate` — Cascades truth changes to dependents after retraction or assertion.
- [file] `reasons/network.py` — Read beyond line 500, especially around line 982, to see the shared `SL`/`CP` handling.
- [file] `tests/test_network.py` — Search beyond line 400 for CP-specific tests that encode expected behavior.
- [file] `reasons/api.py` — Confirm how `--cp` antecedents and outlists are converted into `Justification(type="CP")`.

## Beliefs

- `cp-is-a-justification-type` — CP is represented as `Justification(type="CP")` and is accepted by the API, CLI, DB schema, and network code.
- `cp-doc-rule-is-consistency` — `reasons/__init__.py:13` states that a CP node is `IN` iff its assumptions are consistent.
- `network-add-node-delegates-truth-to-compute-truth` — `Network.add_node` calls `_compute_truth` for nodes with justifications; nodes without justifications become `IN` premises.
- `network-line-982-handles-sl-and-cp` — `reasons/network.py:982` contains a shared `if j.type in ("SL", "CP")` branch.
- `retract-and-assert-use-propagate` — `Network.retract` and `Network.assert_node` update the target node and call `_propagate` to recompute dependents.
