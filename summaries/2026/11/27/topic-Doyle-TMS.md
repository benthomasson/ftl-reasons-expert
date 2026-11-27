---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Background on justification-based truth maintenance systems and non-monotonic “believe unless” reasoning.

## Background: Justification-Based Truth Maintenance and "Believe Unless" Reasoning

### 1. The core JTMS data model

The `reasons` package implements a **Justification-based Truth Maintenance System (JTMS)**. Two SQLite tables in `reasons/storage.py` define the shape of the whole system:

- `nodes` — each with `id`, `text`, a `truth_value` of `IN`/`OUT`, an optional `supporting_justification`, plus provenance
- `justifications` — each with `node_id`, `type`, `antecedents_json`, `outlist_json`, `label`, `content_hash`

These map to the `Justification` dataclass declared at `reasons/__init__.py:7`, whose fields (`type`, `antecedents`, `outlist`, `label`, `content_hash`) mirror the persisted columns. The storage module docstring notes that ACID transactions are used so that **propagation cascades are atomic** — a cascade either completes fully or rolls back, which matters once retractions start cascading.

### 2. Premises vs. justified nodes

`SKILL.md:31-32` gives the base vocabulary:

- A **premise** is a node with *no* justifications and is **IN by default** — created by `reasons add` without `--sl` or `--cp`.
- A **justified node** is IN only while at least one of its justifications is valid; when that justification becomes invalid the node goes OUT.

That inversion is important: in a JTMS, "believed" is the *default* for premises, and derived nodes fall out automatically when support is lost. This is what lets the system maintain consistency without explicit "unassert" calls.

### 3. SL justifications — the monotonic backbone

`SKILL.md:33` — **SL (Support List)** justifications make a node IN when **all** antecedents are IN. This is the primary justification type and is wired through:

- `reasons/api.py:406` — `add_justification(...)`
- `reasons/network.py:496` — `Network.add_justification(...)`
- `reasons/mcp_server.py:204` — `add_justification(node_id, sl, unless, label)`

SL is conjunctive — every antecedent must hold. `CHANGELOG.md:25` records the `--any` flag that lets callers add **one SL per premise** instead of a single conjunctive SL, which is how OR-of-justifications is expressed at a higher level.

### 4. Multiple justifications = OR of ANDs

`SKILL.md:34` — a node can carry multiple justifications and stays IN if **any one** is valid. `tests/test_defeat_justification.py::test_defeat_multiple_justifications` makes this concrete: with two justifications on `b`, defeating index 0 leaves `b` IN; defeating index 1 then drops it to OUT. Combined with SL semantics, the rule is effectively `IN ⟺ (∃ j. ∀ a ∈ j.antecedents. a.IN)` — an OR over justifications, each an AND of antecedents.

### 5. The outlist — where non-monotonicity enters

`SKILL.md:37` is the crux of "believe unless":

> **Outlist** (non-monotonic): A justification can require nodes to be **OUT**. "Believe X unless Y" — if Y goes IN, X goes OUT.

Mechanically, a justification's validity condition becomes **`(∀ a ∈ antecedents. a.IN) ∧ (∀ o ∈ outlist. o.OUT)`**. This breaks monotonicity: adding a new belief (an outlist node becoming IN) can *retract* a previously-justified conclusion. In classic monotonic logic, new facts can only extend what's derivable; here they can also remove conclusions.

The prompt constructed in `reasons/derive.py` (see the `"## Background"` block near the top of the module) explicitly documents a third node class — **outlist-gated conclusions** — and states: *"the conclusion is OUT while the outlist node is IN, and flips IN when it goes OUT."* `parse_proposals` in the same file parses a `### GATE` block with `- Unless: <ids>` and stores them in `justifications.outlist_json`, so gates flow through the same machinery as SL.

### 6. Defeaters — outlist in practice

`reasons/api.py:1322` provides `defeat_justification`, and the accompanying tests in `tests/test_defeat_justification.py` show the runtime flow:

1. A **defeater node** is created (default id `over-generalizes-belief-j0`, or a caller-supplied `defeater_id`) with metadata `defeater_type`, `defeats_node`, `defeats_justification`.
2. The defeater id is added to the target justification's `outlist`.
3. The target node flips OUT; dependents cascade.
4. **Retracting the defeater restores the target to IN** — see `test_defeat_restoration_on_retract`.

That restore step is precisely non-monotonic reasoning: withdraw the objection, and the previously-suppressed conclusion becomes justified again — no manual re-assertion required.

`tests/test_classify_defeaters.py` enumerates `DEFEAT_REASON_TYPES` (7 values) — including `invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by`, `unsupported-conjunct`, `scope-mismatch`, `migrated-retraction`. These are the classic forms a defeater can take: the justification is not *false*, just not sufficient for the current scope.

### 7. Retraction cascades and restoration

`SKILL.md:35` — when a node goes OUT, all dependents whose justifications become invalid go OUT automatically. `test_defeat_cascade_to_dependents` in `tests/test_defeat_justification.py` demonstrates the chain `p → b1 → b2`: defeating `b1`'s justification drops *both* `b1` and `b2`.

The reverse cascade also works (see `SKILL.md:189-197`): a node whose justification becomes valid again is restored to IN, and its dependents re-derive. The dependents index used by the cascade is rebuilt from justifications in `Storage.load` (`reasons/storage.py`) via `network._rebuild_dependents()`, so it is a derived view over the justification graph, not a separate source of truth.

### 8. Contradictions, nogoods, and backtracking

`reasons/contradictions.py` runs an LLM over IN beliefs to produce **NOGOODs** — sets of beliefs that cannot all be true (`### NOGOOD short-kebab-id` with `Claims:` and `Severity:`). Once a nogood is asserted, `Network.add_nogood` (exercised via `tests/test_backtracking.py`) does dependency-directed backtracking:

- `trace_assumptions` walks justification edges back to premises.
- `find_culprits` returns the premises that support each claim, sorted by number of dependents.
- `_entrenchment` picks the *weakest* culprit to retract: the tests in `TestEntrenchment` verify that *sourced* premises, `OBSERVATION` `beliefs_type`, and high-dependent nodes are more entrenched than unsourced, `DERIVED`, or isolated ones. The "physics-speculation bug" test (`test_nogood_retracts_speculation_not_evidence`) encodes this policy directly.

The `backtrack` log entry (`test_backtracking_logs_culprit`) records which premise the system blamed, and `test_inactive_nogood_no_backtracking` confirms no backtracking is done when one member is already OUT — the conflict is already resolved.

### 9. What is not visible in these observations

A few pieces are referenced but their source isn't in the snippets:

- The **propagation algorithm** behind `reasons propagate` (`SKILL.md:132`) — i.e., how recomputation walks the dependents index and decides IN/OUT per node.
- The `Justification` class body in `reasons/__init__.py` (only the line 7 marker is shown).
- Whether `CP` (the second antecedent-based justification type created via `--cp`) is currently used or historical.
- The implementation of `_entrenchment` in `reasons/network.py` (tests exist, source not shown).

To fully answer "how propagation reaches a fixed point after a cascade," those would need to be read.

## Topics to Explore

- [file] `reasons/network.py` — Contains `add_justification` (line 496), `_rebuild_dependents`, `_entrenchment`, `trace_assumptions`, `find_culprits`, and `add_nogood`; the propagation/cascade algorithm lives here.
- [function] `reasons/api.py:defeat_justification` — The public entry point that creates defeater nodes and appends them to the target justification's outlist.
- [function] `reasons/api.py:add_nogood` — Ties the contradiction detector to dependency-directed backtracking and entrenchment-based culprit selection.
- [file] `tests/test_outlist.py` — Exercises outlist semantics in isolation, including `TestOutlistWithMultipleJustifications`, which shows how "unless" interacts with OR-of-justifications.
- [general] `defeater-taxonomy` — The seven `DEFEAT_REASON_TYPES` in `tests/test_classify_defeaters.py` — useful to understand what kinds of justifications are considered defeasible vs. outright wrong.

## Beliefs

- `premise-defaults-to-IN` — A node with no justifications is a premise and is IN by default; only justified nodes can be flipped OUT (`SKILL.md:31`).
- `sl-requires-all-antecedents-IN` — An SL justification is valid if and only if every antecedent in `antecedents_json` is IN (`SKILL.md:33`).
- `node-IN-if-any-justification-valid` — Multiple justifications are logically OR'd: the node stays IN until every justification is invalidated (`SKILL.md:34`, `test_defeat_multiple_justifications`).
- `outlist-makes-reasoning-non-monotonic` — An outlist entry inverts the truth condition: the justified node is IN only while the outlist node is OUT, so an outlist node becoming IN retracts the conclusion (`SKILL.md:37`).
- `defeat-enables-restoration` — `defeat_justification` appends the defeater id to the target's `outlist`, and retracting that defeater restores the target to IN (`test_defeat_restoration_on_retract` in `tests/test_defeat_justification.py`).
