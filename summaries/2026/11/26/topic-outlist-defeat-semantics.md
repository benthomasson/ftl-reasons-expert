---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `Justification.outlist` nodes gate a belief and interact with recompute, challenges, and defeaters.

# How `Justification.outlist` gates a belief

## The data model

`Justification` is a dataclass defined at `reasons/__init__.py:7`. Its `outlist` field (`reasons/__init__.py:21`) is documented as “must be OUT” — a list of node IDs that must **all** be OUT for this justification to hold. The class docstring spells out the two-sided contract:

- **inlist** (antecedents): “AND ALL outlist nodes are OUT” (`reasons/__init__.py:12`)
- **non-monotonic semantics**: “believe X unless Y is…” (`reasons/__init__.py:15`)

At the API level this surfaces as the `unless=` parameter whose docstring is “Comma-separated outlist IDs (must be OUT for justification to hold)” (`reasons/api.py:302`, `:423`). It is parsed (`reasons/api.py:320`, `:437`), namespace-resolved (`reasons/api.py:368`, `:455`), and attached to the constructed `Justification` (e.g. `reasons/api.py:326`, `:334`, `:460`, `:466`).

**Gating semantics** (backed by `tests/test_outlist.py`):

| Situation | Result | Test |
|---|---|---|
| All outlist entries OUT / absent | justification holds | `test_outlist_valid_when_out`, `test_outlist_node_not_in_network` |
| Any outlist entry IN | justification fails | `test_outlist_invalid_when_in` |
| Outlist + inlist combined | both must be satisfied | `test_outlist_with_inlist`, `test_outlist_with_inlist_both_fail` |
| Multiple outlist entries | treated as an AND (all must be OUT) | `test_multiple_outlist_nodes` |
| Another justification still holds | belief stays IN | `test_alternate_justification_survives_outlist`, `test_defeat_multiple_justifications` |

Absent outlist nodes are implicitly OUT (an outlist entry pointing at a nonexistent node behaves as “not believed”). Outlist entries therefore cannot directly *promote* a belief — they only act as a gate.

## Recompute / propagation

There is **no symbol named `recompute`** in the observed files (`locate_recompute` returned `count: 0`). Instead, truth-value changes propagate as a side effect of `assert_node` / `retract_node`, which return a `changed: list[str]` (visible throughout `tests/test_outlist.py`). The relevant code paths are hinted by comments:

- `reasons/api.py:519` — “(restoration from outlist — gated beliefs whose blocker is removed).”
- `reasons/api.py:586` — “(outlist-gated beliefs that lose their justification when this node goes IN).”
- `reasons/api.py:1590–1592` — the actual runtime gate: `outlist_ok = all(... for o in j.outlist)` inside the truth-value evaluation.

Direction of propagation from the tests:

1. Outlist node goes **IN** → dependent belief goes **OUT** (`test_assert_outlist_node_retracts_dependent`).
2. Outlist node **retracted** → dependent belief may be **restored** (`test_retract_outlist_node_restores_dependent`).
3. Both directions cascade through the antecedent graph (`test_outlist_cascade_through_chain`).

Because the outlist check is a pure predicate over the current truth values, propagation is reversible — the same mechanism both retracts and restores, and `explain` output surfaces it as `outlist` / `violated_outlist` (`test_explain_shows_outlist_on_valid`, `test_explain_shows_violated_outlist`).

## Interaction with challenges

`challenge` (`reasons/api.py:1720–1736`) “creates a challenge node and the target goes OUT,” and its inverse `defend` (`reasons/api.py:1741–1756`) “neutralises the challenge, target restored.” Both operations return a `changed` list, i.e., they drive the same propagation path that outlist does.

However, the observations do **not** show `challenge` appending to a `Justification.outlist`. The challenge path tracks state via `node.metadata["challenges"]` (`reasons/api.py:835`, `:3461`, `:3498`). So challenge is a **sibling mechanism**, not an outlist consumer — the observations are insufficient to say whether `challenge` internally materializes a hidden justification or just vetoes the node directly. This is a gap.

## Interaction with defeaters

`defeat_justification` (`reasons/api.py:1322`) is the cleanest example of outlist being used as a live data structure:

1. It creates a new **defeater premise node** (`reasons/api.py:1386–1389`) that comes in as **IN** by default, annotated with `defeater_type`, `defeats_node`, `defeats_justification` metadata (`:1379–:1384`).
2. It **appends the defeater ID to the target justification’s `outlist`** (`:1392–:1393`).
3. It registers the defeater as a dependent of the defeated belief (`:1394`) so future propagation reaches it.

Effect: the defeater is IN, so the justification’s outlist check fails, so the belief goes OUT — graph-native and, crucially, **reversible**. Every guarantee follows from this:

- Retracting the defeater restores the belief (`test_defeat_restoration_on_retract`) — same mechanism as `test_retract_outlist_node_restores_dependent`.
- If another justification survives, the belief stays IN (`test_defeat_multiple_justifications`).
- Defeating one goal cascades to dependents (`test_defeat_cascade_to_dependents`, via `test_outlist_cascade_through_chain` in the network layer).

Supersession shares this mechanism: `reasons/api.py:1187` says “The old node goes OUT via the outlist mechanism (reversible),” and a second appender appears at `reasons/api.py:1526–1527` — likely the supersede path adding a “superseded-by” node to the old node’s outlist.

## Summary

`Justification.outlist` is the single, uniform mechanism for *retractable gating*:

- **Gate**: `all(node IN for node in antecedents) AND all(node OUT or absent for node in outlist)` — evaluated at `reasons/api.py:1590–1592`.
- **Defeat** = adding a new IN node to an outlist (`reasons/api.py:1392–1393`).
- **Restore** = retracting any outlist node — same propagation engine reverses the flip.
- **Challenges** are a parallel veto mechanism keyed off `metadata["challenges"]`; the observations do not confirm they use outlist, so that interaction should be verified in `network.py`.

## Topics to Explore

- [function] `reasons/api.py:defeat_justification` — The canonical defeater entry point that wires a new node into an existing `Justification.outlist`.
- [file] `reasons/network.py` — The actual propagation engine (`assert_node`/`retract_node`) that re-evaluates `outlist_ok` and produces the `changed` list; not in the observations.
- [function] `reasons/api.py:challenge` — Confirm whether the challenge path uses `Justification.outlist` or only `node.metadata["challenges"]`; the observations leave this ambiguous.
- [function] `reasons/api.py:_recompute` (or the enclosing function around line 1590) — The `outlist_ok = all(...)` evaluation point; identifies exactly when the outlist predicate is consulted.
- [file] `tests/test_outlist.py` — Most complete spec of the gate semantics; use as behavioural documentation when modifying `outlist` handling.

## Beliefs

- `outlist-requires-all-entries-out` — A justification holds only when every node in its `outlist` is `OUT` or absent from the network.
- `outlist-enables-non-monotonic-reasoning` — The outlist field implements Doyle's “believe X unless Y is believed” default logic.
- `defeater-uses-outlist` — `defeat_justification` adds a defeater premise node to the target justification’s `outlist` rather than overwriting the justification (`reasons/api.py:1392–1393`).
- `defeater-retraction-restores-belief` — Retracting a defeater node restores the defeated belief because defeat is implemented as an ordinary graph node in the outlist (`test_defeat_restoration_on_retract`).
- `multiple-justifications-survive-defeat` — Defeating one justification leaves the belief `IN` if any other justification still satisfies its own antecedents and outlist (`test_defeat_multiple_justifications`).
