---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: How an `outlist` entry on a justification propagates `IN`→`OUT` through the graph (needed to understand the cascade tests)

# How an `outlist` entry propagates `IN`→`OUT` through the graph

## 1. The data model: `outlist` lives on the justification

An `outlist` is a field on the `Justification` dataclass declared in `reasons/__init__.py`:

- `class Justification` — `reasons/__init__.py:7`
- `outlist: list[str] = field(default_factory=list)` — `reasons/__init__.py:21`
- Comment defining semantics: *"AND ALL outlist nodes are OUT"* — `reasons/__init__.py:12`
- Comment defining intent: *"believe X unless Y is [IN]"* — `reasons/__init__.py:15`

So an outlist entry is **not an antecedent** — it is the opposite: a node ID that is a *defeater*, i.e. it must evaluate to **OUT** for the justification to stand. That is the whole point of the Doyle-style "SL" (support-list) justification: antecedents must be IN, outlist members must be OUT.

## 2. The validity predicate

The evaluation is a simple boolean conjunction. The observable snippet is in `reasons/api.py`:

```python
# reasons/api.py:1590-1592
outlist_ok = all(
    ...                       # per-entry check of j.outlist[...]'s truth value
    for o in j.outlist
)
```

Read together with the comments in `__init__.py`, the rule is:

> A justification is **valid** iff every antecedent is IN **and** every outlist node is OUT.

Because this is a per-entry check, the presence of *any* outlist node that goes IN flips `outlist_ok` to `False`, invalidating that single justification (see `tests/test_outlist.py::TestOutlistBasic::test_outlist_invalid_when_in` and `test_outlist_with_inlist_both_fail`). A missing node is treated as not-believed, i.e. OUT — `test_outlist_node_not_in_network`.

## 3. Why an outlist change can propagate at all

Two dependency edges exist for every node `X`:

- **Antecedent edges** — `X` depends on each ID in `justification.antecedents` (they must be IN).
- **Outlist edges** — `X` depends on each ID in `justification.outlist` (they must be OUT).

Both edges are *dependencies* in the propagation graph, just with different polarities. Changing any outlist node's truth value therefore makes `X` **stale** and it must be re-evaluated. That staleness is the seed of the cascade.

## 4. The cascade: OUT of the outlist node flips the dependent

The tests in `tests/test_outlist.py` document the required behavior precisely:

- **Outlist node goes OUT→IN** — the justification that gates on it loses validity, so `X` goes OUT.
  `TestOutlistCascading::test_assert_outlist_node_retracts_dependent`: assert `y` → `x` goes OUT, and `x` appears in the returned `changed` set.
- **Outlist node goes IN→OUT** — the justification is repaired, so `X` is restored IN.
  `test_retract_outlist_node_restores_dependent`; the code comment at `reasons/api.py:519` explicitly describes *"restoration from outlist — gated beliefs whose blocker is removed."*
- The mirror comment at `reasons/api.py:586` describes the forward direction: *"outlist-gated beliefs that lose their justification when this node goes IN."*

The `changed` set returned by `assert_node`/`retract` (see the two tests above) is how the caller (and the cascade tests) know a flip actually propagated.

## 5. Transitivity: the cascade continues through antecedents

An outlist node going IN invalidates `X`'s justification → `X` becomes OUT → any node `Z` that lists `X` in its **antecedents** now fails its own justification → `Z` goes OUT, and so on.

```
y (outlist edge) ─► x ─(antecedent edge)─► z
assert y  ⇒  x=OUT  ⇒  z=OUT
```

`TestOutlistCascading::test_outlist_cascade_through_chain` asserts exactly this: after `net.assert_node("y")`, both `x` and `z` are `OUT`. So the propagation is *breadth-first along both edge kinds*, not just direct outlist hits.

## 6. Multiple justifications = OR, outlist = AND-per-justification

`TestOutlistWithMultipleJustifications::test_alternate_justification_survives_outlist` proves the combining rule:

- A node is **IN** if at least **one** justification is valid.
- Each justification requires **all** its outlist entries to be OUT (`test_multiple_outlist_nodes`: `x` unless `y` **or** `z` — asserting just one of them kills the justification, and retracting it brings `x` back).

So the propagation recomputes the whole node's truth value after any single justification changes.

## 7. How the API writes outlists (and how defeat reuses propagation)

- CLI/API `unless=` parsing — `reasons/api.py:302, 320, 437` — comma-split trimmed IDs.
- Justification construction with outlist — `reasons/api.py:326, 328, 331, 334, 460, 466`.
- Namespace resolution of outlist IDs — `reasons/api.py:368, 455` — so namespaced and local references normalize before propagation.
- Serialization includes outlist — `reasons/api.py:899`; structural round-trip survives `Storage.save`/`load` and propagation still works after load (`TestOutlistPersistence::test_round_trip`).
- **Defeat helper** — `reasons/api.py:1331-1393` (`defeat_justification`) and `reasons/api.py:1526-1527`. The docstring at `1334-1335` is explicit: the defeater is appended to `justification.outlist`, *"causing the belief to go OUT via graph-native"* propagation. So defeasibility is implemented **on top of** the same outlist mechanism, not a special-case path.
- `explain()` surfaces violation state via `violated_outlist` (`TestOutlistExplain::test_explain_shows_violated_outlist`).

## 8. What is missing from the observations (important caveat)

The **propagation engine itself is not in the gathered snippets.** Specifically:

- `reasons/network.py` is never shown, so the actual `Network.assert_node` / `Network.retract` / staleness queue / topological re-evaluation loop is unread.
- `locate_propagation_logic` (pattern `propagat|IN|OUT`) returned **0 matches**, and `find_cascade_tests` (pattern `cascade`) returned **0 matches** — so neither the propagation function nor its tests were located by the search.
- The body of the function surrounding `reasons/api.py:1590` (`outlist_ok = all(...)`) is truncated; we know the predicate but not its caller.

The behavior above is therefore inferred from the test contract (`tests/test_outlist.py`) and the surviving snippets in `reasons/api.py`, not from reading the re-evaluation loop. The exact staleness/ordering algorithm (single dependency-ordered pass vs. iterative fixpoint) still needs to be checked in `reasons/network.py`.

## Topics to Explore

- [file] `reasons/network.py` — Contains `Network.assert_node` / `retract` and the actual re-evaluation loop that turns outlist changes into `changed`-set cascades; the propagation algorithm itself is here.
- [function] `reasons/api.py:_is_justification_valid` — The truncated function owning `outlist_ok = all(...)` at `reasons/api.py:1590`; confirm antecedent and outlist conjunction order and missing-node handling.
- [function] `reasons/api.py:defeat_justification` — Shows the "append to outlist" path reused by `defeat` and confirms it relies on the same propagation engine.
- [general] `tms-propagation-ordering` — Whether the engine does a single topological pass or iterates to a fixpoint; matters for cycles in outlist↔antecedent dependencies.
- [file] `tests/test_outlist.py` — Full cascade/restoration/explain assertions are the executable spec for this feature.

## Beliefs

- `justification-outlist-entry-requires-node-out` — A justification is valid only when every ID in `Justification.outlist` is OUT (`reasons/__init__.py:12,21`, checked at `reasons/api.py:1590`).
- `outlist-is-a-defeater-not-an-antecedent` — `outlist` entries are stored separately from `antecedents` and are checked with the opposite polarity (must be OUT rather than IN).
- `defeat-justification-mutates-outlist` — `api.defeat_justification` appends the defeater to `Justification.outlist` (`reasons/api.py:1392-1393`, `1526-1527`) rather than removing antecedents, so defeat reuses the native outlist propagation.
- `outlist-cascade-is-transitive-through-antecedent-edges` — Because a node going OUT invalidates justifications whose antecedents include it, an outlist flip propagates to arbitrary depth (`test_outlist_cascade_through_chain`: `x` and `z` both go OUT).
- `node-in-requires-any-valid-justification` — A node is IN if at least one justification passes (all antecedents IN + all outlist OUT), as shown by `test_alternate_justification_survives_outlist`.
- `persistence-round-trips-outlist` — `Storage.save`/`load` preserves `outlist` and propagation still functions after reload (`TestOutlistPersistence::test_round_trip`).
