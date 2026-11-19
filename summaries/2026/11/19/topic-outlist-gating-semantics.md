---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Understand how `unless`/OUT beliefs flip truth values and why GATE proposals are represented through `unless`.

## Explanation

### Truth values and the justification rule

A belief node has a binary truth value, `IN` or `OUT` (`reasons/__init__.py:35`). A justification has two parts:

- **inlist / antecedents**: nodes that must be `IN`
- **outlist / unless**: nodes that must be `OUT`

The core rule is stated in `reasons/__init__.py:12`: a justification holds only if its antecedents are `IN` **and all outlist nodes are `OUT`**. If any outlist node is `IN`, that justification is defeated. A node remains `IN` only if at least one of its justifications still holds; otherwise it may go `OUT` (`reasons/__init__.py:17`).

The `unless` API parameter is exactly this outlist. `reasons/api.py:302` documents it as “comma-separated outlist IDs (must be OUT for justification to hold)”, and `reasons/api.py:320` parses the string into a list. The CLI exposes the same idea at `reasons/cli.py:3078` and `reasons/cli.py:3377`: “Outlist: comma-separated node IDs that must be OUT”.

There is also an outlist-only form: if there is no inlist but there is an `unless`, the node is a premise that holds by default unless one of the blockers is believed. That is explicitly called out in `reasons/api.py:333`.

### How `unless`/OUT beliefs flip truth values

The important inversion is:

- A normal **antecedent** is a positive dependency: antecedent `IN` helps make the dependent `IN`.
- An **outlist/unless** node is a negative dependency: blocker `OUT` lets the justification hold; blocker `IN` defeats it.

So for a node with one justification like:

```text
gated-conclusion: sl=support-for-gated, unless=blocker-x
```

the logic is effectively:

```text
gated-conclusion is IN iff support-for-gated is IN and blocker-x is OUT
```

That produces the flip:

- `blocker-x` is `OUT` → the `unless` condition is satisfied → `gated-conclusion` can be `IN`.
- `blocker-x` becomes `IN` → the `unless` condition fails → that justification is defeated → `gated-conclusion` may go `OUT` if no other justification supports it.
- `blocker-x` is retracted back to `OUT` → the justification can hold again → `gated-conclusion` may be restored to `IN`.

This is the non-monotonic behavior described in `reasons/__init__.py:15-17`: adding a belief can force another belief `OUT`, and removing/retracting it can bring the original belief back. The API tracks these flips as “went OUT” and “went IN” changes in `reasons/api.py:490-491`, and `reasons/api.py:519` explicitly mentions restoration from outlist when a blocker is removed.

A concrete test fixture shows the pattern: in `tests/test_report_gated.py`, `gated-conclusion` is created with `sl="support-for-gated", unless="blocker-x"`. Because `blocker-x` is `IN`, it gates `gated-conclusion`, and `api.list_gated` / `api.report_gated` report that blocker as an active blocker. Conversely, in `tests/test_propose_update.py`, `derived-c` has outlist `["bad-weather-warning"]` while `bad-weather-warning` is `OUT`, so the belief is `IN`.

### Why GATE/gated proposals are represented through `unless`

A gate is naturally an **exception or blocker**, not a positive antecedent. If you encoded “X is gated by Y” as a positive antecedent, you would have to say “believe X when Y is satisfied,” which is the wrong direction for an open issue or blocker. The TMS representation wants:

> believe X unless blocker Y is believed/IN

That maps directly to the outlist/`unless` mechanism:

- create or identify a blocker node `Y`
- attach `Y` to the target’s justification outlist
- as long as `Y` is `OUT`, the target can remain `IN`
- when `Y` becomes `IN`, the target’s justification is defeated and the target can go `OUT`

This is why gated beliefs are grouped by their outlist blocker. `reasons/api.py:3628` defines `list_gated`, which returns blockers and the beliefs they gate; `reasons/api.py:3635-3636` documents the result as `{blocker_id: {"text": ..., "gated": [...]}}`; and `reasons/api.py:3667-3668` counts total gated beliefs. `reasons/report_gated.py` then builds a report of active blockers and fixed defects from this representation.

The proposal/update path also preserves this structure. `reasons/propose_update.py` formats outlists as “Unless (must be OUT):” and includes the blocker IDs and texts so proposals/LLM review see the gate relation. `reasons/api.py:5695` reads `unless` from a proposed config, which is consistent with proposals carrying the gate via outlist rather than as a positive antecedent. `reasons/derive.py:57` also contains guidance that outlist-gated beliefs should have a positive antecedent and an `unless` part.

One caveat: the supplied observations do not include the full truth-maintenance propagation algorithm or a fully explicit “GATE proposal” schema. The observable representation is that gating is encoded as outlist/`unless`; the exact proposal object shape is only partially visible through `api.py:5695`.

## Topics to Explore

- [file] `reasons/__init__.py` — Defines the `Justification` outlist/antecedent model and the `IN`/`OUT` truth-value semantics.
- [function] `reasons/api.py:add_node` — Shows how `unless` becomes an outlist and how outlist-only justifications are constructed.
- [function] `reasons/api.py:list_gated` — Groups active blockers with the beliefs they gate.
- [file] `reasons/derive.py` — Contains guidance and parsing for outlist-gated belief generation.
- [general] `truth-maintenance-propagation` — The full recompute/fixpoint algorithm is not shown in these observations; inspect it to confirm ordering and multiple-justification behavior.

## Beliefs

- `justification-requires-outlist-out` — A justification with an outlist holds only when every node in that outlist has `truth_value == "OUT"`.
- `blocker-in-causes-gated-belief-out` — If an outlist blocker is `IN` and no alternative justification remains, the gated belief can flip from `IN` to `OUT`.
- `retracting-blocker-enables-restoration` — Moving an outlist blocker from `IN` to `OUT` can restore an outlist-gated belief to `IN` during propagation.
- `gate-relationships-map-to-unless` — Gated-belief relationships are represented by storing blocker node IDs in the target justification’s `unless`/outlist.
- `outlist-only-justification-is-default-belief` — An `unless`-only justification represents a premise that holds by default while its blockers are `OUT`, as supported by `reasons/api.py:333`.
