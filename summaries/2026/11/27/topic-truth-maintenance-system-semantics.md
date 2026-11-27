---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Background on SL/CP justifications, outlist defeaters, and premise-based TMS reasoning.

## Background: SL/CP Justifications, Outlist Defeaters, and Premise-Based TMS Reasoning

This codebase implements a **Doyle-style Truth Maintenance System (TMS)**: nodes carry a truth value (`IN` / `OUT`) and are justified by explicit records rather than by mutation of the node text. The interesting part is how non-monotonic belief revision is layered on top via the **outlist**.

### 1. The unit of belief: premises vs. derived nodes

Every node is either:

- **A premise** — has *no* `justifications`. `reasons/__init__.py:31` states: *"A premise has no justifications and is IN by default."* `reasons/api.py:830` mirrors this in the list logic (`is_premise = not node.justifications`).
- **A derived node** — has one or more `Justification` records. It is `IN` iff at least one of its justifications is *valid* (see below).

This is the "premise-based" foundation: everything that is believed is either explicitly assumed or inferred from something that is assumed, transitively (verified by `reasons/api.py:1056` `trace_premises`, which walks justification chains back to nodes with no justifications).

### 2. The `Justification` record and the truth condition

From `reasons/__init__.py:12–21`:

```
AND ALL outlist nodes are OUT
...
outlist: list[str] = field(default_factory=list)  # must be OUT
```

A single justification carries:

- `type` — `"SL"` or `"CP"`
- `antecedents` — the *inlist*; these must all be `IN`
- `outlist` — these must all be `OUT`
- optional `label`

**Truth condition (SL):** *all antecedents IN* **AND** *all outlist nodes OUT*. A node is `IN` iff **any one** of its justifications satisfies this.

The SL semantics are exercised directly in `tests/test_outlist.py`:

- `Justification(type="SL", antecedents=[], outlist=["y"])` → `x` is `IN` when `y` is `OUT` (`test_outlist_valid_when_out`), `OUT` when `y` is `IN` (`test_outlist_invalid_when_in`).
- `Justification(type="SL", antecedents=["a"], outlist=["y"])` → `x` requires `a IN` and `y OUT` (`test_outlist_with_inlist`, `test_outlist_with_inlist_both_fail`).
- Absent outlist nodes are treated as `OUT` (`test_outlist_node_not_in_network`).

`reasons/api.py:1590–1592` is the runtime check:

```python
outlist_ok = all(
    ...
    for o in j.outlist
)
```

(presumably `node(o).truth_value == "OUT"`; the middle line was truncated in the excerpts, but the shape is clear).

### 3. SL vs CP

`reasons/api.py:326–334` constructs both:

```python
justifications.append(Justification(type="SL", antecedents=[a], outlist=outlist, label=label))
...
justifications.append(Justification(type="SL", antecedents=antecedents, outlist=outlist, label=label))
...
justifications.append(Justification(type="CP", antecedents=antecedents, outlist=outlist, label=label))
```

So both types share the same `antecedents` + `outlist` payload. The naming reflects Doyle's classic distinction — **SL** (Support List, the standard conjunctive rule) vs. **CP** (Conditional Proof, typically a weaker/locally-assumed rule). **Caveat:** the observations do **not** contain a test or the derive-loop body demonstrating how `CP` differs semantically from `SL`, nor the default branch at `api.py:332` (`elif outlist:`) which treats an `unless`-only add as a bare SL. This is the biggest gap in the current evidence; treat `SL` as "and-of-antecedents + not-any-of-outlist" and treat `CP` as needing further reading.

### 4. Outlist: the non-monotonic gate

The outlist is the mechanism for **non-monotonic reasoning** — the classic "believe X *unless* Y." The user-facing knob is `--unless`:

- `reasons/api.py:302, 320` — parses comma-separated IDs into `outlist`.
- `CHANGELOG.md:131` — "Non-monotonic reasoning via outlist (`believe X unless Y`)".
- `SKILL.md:56` — same framing.

Propagation is bidirectional and cascades (see `tests/test_outlist.py`):

- Asserting an outlist node (`test_assert_outlist_node_retracts_dependent`) → dependent goes `OUT`.
- Retracting it (`test_retract_outlist_node_restores_dependent`) → dependent returns to `IN`.
- Chain cascade (`test_outlist_cascade_through_chain`) → dependents-of-dependents also flip.
- Multiple outlist nodes must **all** be `OUT` (`test_multiple_outlist_nodes`).
- Durable across SQLite round-trip (`test_round_trip`).

Because outlist entries live *on the justification record* (not in code), this is the "outlist defeater" substrate that the higher-level commands manipulate.

### 5. Defeaters — outsourcing the outlist

Instead of mutating a justification's outlist in place, the API offers `defeat_justification` (`reasons/api.py:1326–1408`). From the docstring at line 1331:

> *"Defeat a justification by adding a defeater belief to its outlist. Creates a defeater belief (premise node) and adds it to the specified justification's outlist, causing the belief to go OUT via graph-native..."*

Mechanics (`api.py:1371–1394`):

1. Generate a `defeater_id` like `"{type}-{node_id}-j{index}"` (line 1372).
2. `net.add_node(id=defeater_id, ...)` — the defeater is a **premise**, so it's `IN` by default.
3. `justification.outlist.append(defeater_id)` (line 1393).
4. `defeater.dependents.add(node_id)` for provenance.

The net effect is that the *outlist is satisfied only while the defeater is OUT*, so the target truth value flips `IN → OUT` immediately. Because the defeater is a first-class belief, users can later retract it to *undo* the defeat — no code change required. This is the "graph-native" replacement for ad-hoc mutation.

`reasons/api.py:1419–1529` provides a more sophisticated variant (`defeat_with_scope`): it creates *scope beliefs* (premises describing what each antecedent actually establishes) plus a **derived** defeater. When scope beliefs are retracted, the derived defeater goes `OUT`, its own antecedents are gone, and the original belief is restored — a truly non-monotonic cycle that pure SL-only TMS couldn't express.

`CHANGELOG.md:33` notes a related design tweak: `import-beliefs` moved the "active premise" from the antecedent list *into the outlist* so individual beliefs become retractable independently — a concrete example of using outlist gates for modularity rather than defeating.

### 6. Dialectical layer: challenge / defend

`tests/test_dialectical.py` documents Doyle §6 in user-facing form, implemented in `reasons/network.py` (referenced but not quoted in the observations) and exposed in `reasons/api.py`:

- `net.challenge("a", "reason")` creates a `challenge-a` node (`IN` by default) and adds it to `a`'s outlist, so `a` goes `OUT` and cascades (see `test_challenge_premise`, `test_challenge_cascades`).
- `net.defend(...)` creates a `defense-*` node and puts it in the *challenge's* outlist, flipping the challenge `OUT` and restoring `a` (`test_defend_restores_target`).
- This nests arbitrarily — challenge-the-defense, defend-the-defense, etc. (`test_three_level_chain`).

Every step is again just outlist bookkeeping — this is why challenge/defend and `defeat_justification` share a conceptual substrate.

`SKILL.md:91` documents the user-facing form:
> *"Run `reasons challenge TARGET \"reason\"`. Creates a challenge node (IN by default) and adds it to the target's outlist."*

### 7. Nogood and dependency-directed backtracking

`SKILL.md:38` and `SKILL.md:99` describe the contradiction handler:

> *"A set of nodes that cannot all be IN simultaneously. When detected, dependency-directed backtracking trace..."*

The full backtracking implementation is not shown in the observations (it would live in `reasons/network.py` or `reasons/derive.py`), but the contract is clear from the docstring: given a nogood set, walk the justification chains (the same `trace_premises` machinery at `api.py:1056`) to find which premise is the "culprit," and retract it, causing a controlled cascade rather than an arbitrary one.

### 8. Premise manipulation as a first-class operation

Because premises are simply "nodes without justifications," several commands exist to convert beliefs into (or out of) that state:

- `convert-to-premise` (`SKILL.md:104–109`) strips all justifications from a node, making it `IN` by default. The docstring explicitly warns: *"`reasons assert` alone is NOT sufficient ... it marks the node IN but doesn't remove the SL justification."*
- `reasons list --premises` (`SKILL.md:145`) rounds up the set of foundation nodes.
- `reasons trace <node>` (`SKILL.md:251`) reports which premises a conclusion rests on.

Together these enforce the "premise-based" invariant: every `IN` derived node has a finite chain of `SL/CP` justifications terminating in premises.

### 9. Summary mental model

| Concept | Record | Flipped by |
|---|---|---|
| Premise | `Node.justifications == []` | `retract` / `challenge` / `defeat` |
| SL / CP rule | `Justification(antecedents, outlist)` | Removing an antecedent (`IN→OUT`) or asserting an outlist node |
| Outlist defeater | `Justification.outlist` entry | Asserting the defeater (defeat) or retracting it (restoration) |
| Defeater command | `defeat_justification` | Creates a premise + appends to outlist |
| Challenge | `challenge-*` node on outlist | Defend it, or retract the challenge |
| Nogood | ad-hoc set of nodes | Directed backtracking over `trace_premises` |

The system stays honest because **every belief-revision action is expressed as an outlist/antecedent mutation**, so truth-value propagation, persistence (`Storage`), and provenance (`explain`) all observe the same graph.

### What's not in the observations

- The actual **derive/truth-propagation loop** is elided — `reasons_lib/derive.py` and `reasons_lib/storage.py` are one-line re-export shims pointing at `reasons.derive` / `reasons.storage`, and neither body is quoted.
- The **semantic difference between `SL` and `CP`** is not demonstrated (no CP tests in `test_outlist.py` or `test_dialectical.py`).
- The **nogood backtracking algorithm** (`SKILL.md:99`) is described but not shown.

Reading `reasons/network.py` and `reasons/derive.py` directly would close these gaps.

## Topics to Explore

- [function] `reasons/api.py:defeat_justification` — The exact construction of a premise defeater and the append-to-outlist line (`api.py:1392–1394`); the docstring explains "graph-native" defeat.
- [function] `reasons/api.py:defeat_with_scope` — Creates scope beliefs + a *derived* defeater, showing the more sophisticated non-monotonic restoration path (`api.py:1419–1529`).
- [file] `reasons/network.py` — Contains the actual truth-propagation loop, `challenge`/`defend` implementations, and likely the nogood backtracking; the observations only show tests calling into it.
- [file] `reasons/derive.py` — Where the SL vs. CP truth condition and outlist check (`api.py:1590`) are actually executed; needed to nail down the SL/CP distinction.
- [general] `sl-vs-cp-semantics` — Search the derive loop and any CP-only tests to determine what `type="CP"` means beyond `type="SL"` (the CHANGELOG and SKILL.md don't say).
- [file] `.claude/skills/reasons/SKILL.md` — The user-facing contract for every command (`challenge`, `defend`, `convert-to-premise`, `nogood`, `trace`); useful as a spec when reading the implementation.

## Beliefs

- `outlist-requires-all-nodes-out` — A justification is valid only if every node in its `outlist` currently has truth value `OUT`.
- `premise-is-a-node-with-no-justifications` — A node with an empty `justifications` list is a premise and defaults to `IN`.
- `sl-and-cp-share-antecedent-outlist-shape` — Both `SL` and `CP` justifications carry identical `antecedents` and `outlist` fields, differing only in the `type` string.
- `defeat-justification-appends-defeater-to-outlist` — `defeat_justification` creates a premise defeater node and appends its ID to the target justification's `outlist`, flipping the target `OUT` until the defeater is retracted.
- `challenge-creates-outlist-defeater` — `net.challenge(target, ...)` creates a `challenge-<target>` node (IN by default) and adds it to the target's outlist, which is why the target goes `OUT` and cascades.
- `defend-neutralizes-challenge-via-outlist` — `net.defend(target, challenge, ...)` creates a `defense-<challenge>` node that is added to the *challenge's* outlist, restoring the target by making the challenge `OUT`.
- `trace-premises-returns-foundation-nodes` — `api.trace_premises` (around `api.py:1056`) walks justification chains and returns only nodes with no justifications, filtered by `visible_to`.
- `unless-flag-populates-outlist` — The `--unless` CLI flag (`api.py:302, 320`) parses a comma-separated ID list directly into the created justification's `outlist`.
