---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Background on Doyle's Truth Maintenance System and why the inlist/outlist asymmetry exists.

# Background on Doyle's Truth Maintenance System and why the inlist/outlist asymmetry exists

## 1. What the system is

`reasons` is explicitly built on Doyle's 1979 paper — see `reasons/__init__.py:1` ("data model based on Doyle (1979)"), `reasons/network.py:3` ("Implements Doyle's (1979) TMS algorithm"), and `README.md:3`/`CLAUDE.md:5`. The reference is `Doyle, J. (1979). A Truth Maintenance System. Artificial Intelligence, 12(3), 231–272` (`README.md:198`).

The unit of belief is a *node* holding a truth value (`IN` = believed, `OUT` = not believed) and one or more *justifications*. A node is `IN` when **at least one** of its justifications is valid. `justifications` records why — the only justification type shown in the observations is `SL` (support-list), plus `CP` referenced at `reasons/api.py:331`. The `Justification` class itself is defined in `reasons/__init__.py:7` and carries:

- `antecedents` — the **inlist** (`__init__.py:20`): every ID here must be `IN`.
- `outlist` — the **outlist** (`__init__.py:21`): every ID here must be `OUT`.
- plus `type`, `label`, `content_hash` (used by `reasons/storage.py:save` when writing the `justifications` table; `outlist_json` is a first-class column, not an afterthought).

`reasons/__init__.py:11-12` states the validity rule plainly: a node is `IN` iff **ALL** antecedents (inlist) are `IN` **AND ALL** outlist nodes are `OUT`. Note this is a conjunction across both lists, not "either" — `tests/test_outlist.py::test_outlist_with_inlist_both_fail` confirms it: if an antecedent is `OUT`, the justification fails regardless of the outlist. The conjunction is what makes the two lists *and*-ed per justification; multiple justifications are then *or*-ed at the node level (`test_alternate_justification_survives_outlist`).

## 2. Why there is an inlist AND an outlist — the asymmetry argument

The core reason is **non-monotonicity**. `reasons/__init__.py:15` and `README.md:15` both paraphrase the intent: *"believe X unless Y is believed."*

If the data model had **only** an inlist (i.e., antecedents that must be `IN`), you could never *retract* a belief by *adding* a belief. Adding nodes would only ever satisfy more antecedents, so the set of `IN` nodes would grow monotonically. That is fine for pure forward-chaining logic, but it does not model defeasible reasoning — default assumptions, diagnosis hypotheses, evidence that overrides speculation, or dialectical argumentation (see `tests/test_dialectical.py:3`, "Doyle Section 6: arguments become explicit, challengeable beliefs").

Doyle's key innovation (`tests/test_outlist.py:3`) is that a justification is valid only if:

1. every inlist node is *present and IN*, and
2. every outlist node is *absent or OUT*.

Condition (2) is what makes the system non-monotonic: **asserting a node that appears in somebody's outlist invalidates that justification** — even though nothing about that justification's antecedents changed. `tests/test_outlist.py::test_assert_outlist_node_retracts_dependent` is the canonical example: `X unless Y`; assert `Y` → `X` flips to `OUT`.

The asymmetry is therefore structural, not stylistic:

- The **inlist** expresses *positive support* — "derive X from these." Making one of them `IN` is what *creates* the justification. Making one `OUT` *destroys* it.
- The **outlist** expresses *negative evidence / defeat* — "X holds as long as no counter-evidence is believed." Making one `OUT` is what *creates* the justification; making one `IN` *destroys* it.

Same two lists, opposite truth-value semantics. If they were merged into a single "supports" list you would lose the ability to distinguish "I need A" from "I need not-A," and non-monotonic retraction by assertion would be impossible. `reasons/api.py:333` names this exactly: an outlist-only justification is "a premise that holds unless something is believed."

## 3. Where the asymmetry shows up in the code

### Data model
- `reasons/__init__.py:20-21` — two separate fields on `Justification`.
- `reasons/storage.py` — the `justifications` table has both `antecedents_json` and `outlist_json` columns; `save`/`load` preserve both independently (see `save` and `load` in that file, and `test_outlist.py::TestOutlistPersistence::test_round_trip` verifying the round-trip).

### Propagation semantics
- `tests/test_outlist.py::test_outlist_valid_when_out` — Y is `OUT`, so X is `IN`.
- `tests/test_outlist.py::test_outlist_invalid_when_in` — Y is `IN`, so X is `OUT`. Same list, opposite direction from an inlist.
- `tests/test_outlist.py::test_outlist_node_not_in_network` — a referenced outlist node that doesn't exist in the network is treated as `OUT` ("absent = not believed"). This is the *negation-as-failure* flavor of the outlist. An inlist node that doesn't exist can't be `IN`, so the parallel rule for inlist is the mirror image.
- `tests/test_outlist.py::test_multiple_outlist_nodes` — all outlist nodes must be `OUT`; asserting *just one* is enough to retract.

### API surface
- `reasons/api.py:302, 423` — the `--unless` CLI parameter is documented as "Comma-separated outlist IDs (must be OUT for justification to hold)." The inlist gets its own parameter (`--sl`). They are separate precisely because they mean different things.
- `reasons/api.py:326-334` — when `--unless` is provided but no `--sl`, the builder creates `Justification(type="SL", antecedents=[], outlist=outlist, …)` — a defeasible premise with an empty inlist. When both are given, it builds `SL(antecedents, outlist)`.
- `reasons/api.py:1331-1335` — the `defeat` command: "Defeat a justification by adding a defeater belief to its outlist," and `api.py:1392-1393` appends the defeater ID to `justification.outlist`. This is the outlist asymmetry exercised through the API — a defeater is a *node* whose mere existence (being `IN`) invalidates someone else's justification.
- `reasons/api.py:1187` — `supersede` uses the same mechanism: "The old node goes OUT via the outlist mechanism (reversible)." So both defeating and superseding ride on the outlist rather than on a separate "retracted by" flag.

### Restoration — the payoff of the asymmetry
- `CLAUDE.md:95` — "Retracted nodes stay in DB (Doyle's insight: enables restoration without rederivation)."
- `tests/test_outlist.py::test_retract_outlist_node_restores_dependent` — retract the defeater Y and X comes back `IN`, *without recomputing X's derivation*. This is the reason the outlist is a list of *node IDs* rather than an inline boolean: the TMS can recompute dependents by flipping truth values, and the outlist dependency (`Y → X` for retraction purposes) must be tracked just like an inlist dependency, even though the *sign* of the dependency is reversed (asserting Y retracts X, whereas asserting an antecedent asserts X).
- `tests/test_outlist.py::test_outlist_with_multiple_justifications` / `test_alternate_justification_survives_outlist` — because justifications are OR-ed, the outlist's effect is per-justification, not per-node. That's what allows a delayed retraction until *all* justifications (including any inlist-only ones) fail.

## 4. What the observations don't cover

A few things worth flagging as *not* directly answered by the snapshot:

- The actual propagation algorithm in `reasons/network.py` isn't in the observations — only the file's docstring. We can *infer* that the dependent index must track outlist edges too (from `reasons/storage.py:load`, which calls `network._rebuild_dependents()` after rehydrating justifications), but the specific cycle-handling (Doyle's labeling / well-founded order) is not shown.
- Whether "absent outlist node = `OUT`" (shown in `test_outlist_node_not_in_network`) is implemented as "treat missing node as `OUT`" or "treat missing node as not-in-dependents" — either way the observable behavior matches negation-as-failure.
- Doyle's original paper also defines `SUM`/`GF` justifications (abstracted views of sub-derivations), referenced at `tests/test_summarize.py:3`. The observations only show `SL` and `CP` (`reasons/api.py:331`) — worth confirming if `SUM`/`GF` is actually implemented or just sketched.

## Topics to Explore

- [file] `reasons/network.py` — The propagation engine that walks inlist and outlist edges; the truth-value labeling algorithm (Doyle's BE/BR reasoning) lives here.
- [file] `tests/test_outlist.py` — The behavioral spec for the asymmetry: valid-when-out, invalid-when-in, absence semantics, cascading, restoration, and multiple justifications.
- [file] `reasons/api.py` — `add_node` handles `--sl` vs `--unless` at lines 300-370 (and the second builder around 423-470); worth tracing how `justifications` are assembled from CLI args.
- [function] `reasons/api.py:defeat` — Shows the outlist being mutated at runtime to encode a defeat relation between an existing belief and a target justification.
- [general] `Doyle-1979-SL-justifications` — The original paper's SL / SUM / GF taxonomy; useful for deciding whether the remaining justification kinds are implemented or only planned (`tests/test_summarize.py:3`).

## Beliefs

- `sl-justification-is-valid-iff-inlist-all-in-and-outlist-all-out` — A `Justification(type="SL", …)` makes its node `IN` iff every `antecedents` entry is `IN` and every `outlist` entry is `OUT` (`reasons/__init__.py:11-12`, `tests/test_outlist.py`).
- `outlist-enables-non-monotonic-retraction` — Asserting a node that appears in another belief's `outlist` flips that belief to `OUT` without changing any of its antecedents (`tests/test_outlist.py::test_assert_outlist_node_retracts_dependent`).
- `absent-outlist-node-counts-as-out` — A node ID referenced in an `outlist` that does not exist in the network is treated as `OUT`, so the justification still holds (`tests/test_outlist.py::test_outlist_node_not_in_network`).
- `outlist-only-add-builds-empty-inlist-justification` — `api.add_node(..., unless="y")` without `--sl` produces `Justification(type="SL", antecedents=[], outlist=["y"])` (`reasons/api.py:333-334`, `tests/test_outlist.py::test_unless_only_no_sl`).
- `defeat-appends-to-outlist` — `api.defeat(...)` mutates the target justification's `outlist` to include the defeater node ID rather than adding a separate "retracted" marker (`reasons/api.py:1331-1335`, `1392-1393`).
- `storage-persists-outlist-as-json` — The `justifications` table stores `outlist_json` alongside `antecedents_json`, and round-trips both with propagation intact (`reasons/storage.py`, `tests/test_outlist.py::TestOutlistPersistence::test_round_trip`).
