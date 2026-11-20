---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Explore `IN`/`OUT` propagation, `SL` vs `CP` justifications, `outlist` defeat behavior, and dependent cascade rules.

# IN/OUT Propagation, SL vs CP, Outlist Defeat, and Dependent Cascades

This codebase implements a **Reason Maintenance System (RMS)** in the style of Doyle — a non-monotonic belief network where a node's truth value is computed from the validity of its justifications, cascades propagate automatically when dependencies change, and non-monotonicity is expressed through the `outlist` ("must be OUT") mechanism.

## 1. The IN/OUT evaluation model

Each node has a binary `truth_value: "IN" | "OUT"` (`reasons/__init__.py:35`). The core evaluation rule is stated at `reasons/__init__.py:30-31`:

> **A node is IN iff ANY of its justifications is valid. A premise has no justifications and is IN by default.**

So the formula decomposes as:

```
node.truth_value == "IN"
    ⟺ is_premise(node)                                  # no justifications → IN
    OR  ∃ j ∈ node.justifications : valid(j)

valid(j)  ⟺  all(a in j.antecedents : a.truth_value == "IN")   # inlist AND
           AND all(o in j.outlist     : o.truth_value == "OUT")  # outlist AND
```

- **"OR" across justifications** — a single surviving justification keeps the node IN (`tests/test_outlist.py:test_alternate_justification_survives_outlist`).
- **"AND" inside a justification** — every antecedent must be IN and every outlist element must be OUT.
- The outlist check is literally encoded as `outlist_ok = all(...)` at `reasons/api.py:1590-1592`.
- The recurrences are handled by the propagation loop (see §4) and the explain trace at `reasons/api.py:1590`.

A subtle but important invariant: **absent outlist nodes are treated as OUT** (`tests/test_outlist.py:test_outlist_node_not_in_network`). Since "must be OUT" is satisfied when the node isn't even believed, a default justification like "X unless Y" fires whenever Y is absent or retracted.

## 2. `SL` vs `CP` justifications

The `Justification` dataclass (`reasons/__init__.py:7-21`) supports two `type` values:

| type | Meaning | Validity rule |
|------|---------|---------------|
| **SL** | Support List | `∀ a ∈ antecedents : a IN` **AND** `∀ o ∈ outlist : o OUT` (`reasons/__init__.py:11-12`) |
| **CP** | Conditional Proof | `assumptions are consistent` (`reasons/__init__.py:13`) |

**SL** is the normal deductive / default rule — it captures both premises ("A implies X") and defaults ("X unless Y"). **CP** is intended for hypothetical reasoning (assumptions) but its consistency check is not visible in the supplied observations (see Missing §).

How they're constructed by the API (`reasons/api.py:300-334`):

- **`sl="a,b"`** → one `Justification(type="SL", antecedents=["a","b"])` — **all** must be IN (`reasons/api.py:328`).
- **`sl="a,b"` with `any_mode=True`** → separated into one SL per antecedent (`reasons/api.py:326`), i.e. a **disjunctive** support ("a OR b") rather than a conjunctive one. This is the key way OR semantics are expressed.
- **`cp="a,b"`** → one `Justification(type="CP", antecedents=["a","b"])` (`reasons/api.py:331`).
- **`unless="y,z"` alone** → a "premise-like" SL with empty antecedents and the outlist (`reasons/api.py:334`). This directly models "X by default unless Y". Confirmed by `tests/test_outlist.py:test_unless_only_no_sl`, which asserts `result["type"] == "SL"`.
- **No justification and no premise** → the API auto-creates an SL gated on `ns:active` (`reasons/api.py:357-359`), which links every non-premise node to its namespace's activation premise, so retracting the namespace cascades everything OUT (`reasons/api.py:221`, `api.py:1938`).

`add_justification` (`reasons/api.py:406-500`) mirrors this: `jtype` defaults to `"SL"`, and `any_mode` only applies when `jtype == "SL"` with more than one antecedent (`reasons/api.py:457-460`). `remove_justification` at `reasons/api.py:1129` deletes a single justification and re-propagates.

## 3. `outlist` defeat behavior (non-monotonicity)

The docstring at `reasons/__init__.py:15-17` captures the semantics precisely:

> "Believe X unless Y is believed." If Y comes IN, the justification becomes invalid and X may go OUT (if no other justification supports it).

Behavior distilled from `tests/test_outlist.py`:

| Scenario | Outcome |
|----------|---------|
| Y retracted, X's only justification `SL([], outlist=["y"])` | X IN (`test_outlist_valid_when_out`) |
| Y IN | X OUT (`test_outlist_invalid_when_in`) |
| Y retracted, X `SL(["a"], outlist=["y"])`, A IN | X IN (`test_outlist_with_inlist`) |
| A OUT, Y retracted | X OUT (`test_outlist_with_inlist_both_fail`) |
| Y does not exist | Outlist satisfied → X IN (`test_outlist_node_not_in_network`) |
| Outlist `["y","z"]`, both retracted | X IN; asserting **either** flips X OUT (`test_multiple_outlist_nodes`) |
| Backup justification `SL(["a"])` alongside default | Assert Y → still IN, retract A → OUT (`test_alternate_justification_survives_outlist`) |

**Defeat as a graph operation.** Rather than a special "defeats" edge, defeat is modeled by mutating the outlist. `defeat_justification` (`reasons/api.py:1322-1394`):

```
1331  # Defeat a justification by adding a defeater belief to its outlist
1392  if defeater_id not in justification.outlist:
1393      justification.outlist.append(defeater_id)
1394      defeater.dependents.add(node_id)
```

The same pattern repeats for scope challenges at `reasons/api.py:1518-1528` and a related variant at `api.py:1630`. This makes defeat **reversible**: to un-defeat, retract the defeater, which is itself just another node, and normal propagation flips the target IN again (`tests/test_outlist.py:test_retract_outlist_node_restores_dependent`).

**Dialectical reasoning is outlist defeat.** The `tests/test_dialectical.py` module shows that `challenge("a", ...)` creates a `challenge-a` node whose presence in `a`'s outlist knocks `a` OUT, and `defend("a", "challenge-a", ...)` creates a `defense-challenge-a` node whose presence in `challenge-a`'s outlist defeats the challenge, restoring `a`. Chains of challenge/defense/counter-challenge naturally fall out of this pattern (`test_three_level_chain`). The dependents integrity test confirms the graph contract: `assert "target" in net.nodes[cid].dependents` (`tests/test_dependents_integrity.py:test_after_challenge`), i.e. the challenge node has the target as a reverse-dependency because the target cites it in an outlist.

**Explainability of defeat.** `net.explain("x")` exposes both `outlist` (for valid justifications) and `violated_outlist` (for invalid ones) — see `tests/test_outlist.py:TestOutlistExplain`.

**LLM gate proposal.** The `derive.py` prompt (`reasons/derive.py:DERTIVE_PROMPT`) recognizes that `outlist`-gated conclusions ("GATE") are a first-class output of higher-level reasoning — parsed by `parse_proposals` (`reasons/derive.py`) which records the `unless` field and maps it to `Justification.outlist` (per `reasons/api.py:356-368`).

## 4. Dependent cascade rules

The `dependents: set[str]` field (`reasons/__init__.py:38`) is a **reverse index**: if node `X` cites `Y` anywhere in *any* justification, then `Y.dependents` contains `X`. Crucially, this includes **both** antecedent and outlist references:

- `tests/test_dependents_integrity.py:test_outlist_dependency` — a node whose only justification is `SL([], outlist=["a"])` still shows up in `a.dependents`.
- `test_both_antecedent_and_outlist` — both antecedent `a` and outlist `b` acquire the derived node as a dependent.
- `api.py:2579-2582` — `_rebuild_dependents` adds the dependent to both `ant_id` and `out_id` sets, reflecting this dual role.

This means **cascade behavior is symmetric for antecedents and outlists** with respect to *graph traversal*, but opposed in *truth-value direction*:

| Change to Y | X parent uses Y as… | Effect on X |
|-------------|---------------------|-------------|
| Y IN → OUT | antecedent | Justification may fail → X may go OUT |
| Y IN → OUT | outlist | Outlist becomes satisfied → X may go IN (restoration) |
| Y OUT → IN | antecedent | Justification may succeed → X may go IN (restoration) |
| Y OUT → IN | outlist | Outlist violated → X may go OUT (defeat) |

Both directions are exercised in `tests/test_outlist.py:TestOutlistCascading` and in the dialectical tests (`test_challenge_cascades`, `test_defend_cascades_restoration`).

**Cascade mechanics.** Retraction (`reasons/api.py:475-…`) and assertion (`api.py:757`) walk the dependents graph:
- `_cascade_depth(net, target_id, retracted_id)` at `reasons/api.py:737-745` BFS-traverses `current.dependents` to measure how far a retraction propagates.
- `get_status`/`propagate` return the list of affected nodes with `depth` and `dependents` size (`api.py:560-561`, `api.py:618-628`, `api.py:711-720`).
- Propagation history is queryable at `api.py:1799`.

**Consistency invariants.** Because the dependents index can drift, the system provides:

- `verify_dependents()` — read-only check that every reverse entry corresponds to a real forward reference (`tests/test_dependents_integrity.py:TestVerifyDependents`).
- `_rebuild_dependents()` — canonical, idempotent rebuild that clears and re-derives dependents from justifications (`test_single_dependency`, `test_rebuild_idempotent`, `test_rebuild_clears_stale_entries`).
- Every mutation (`add_node`, `add_justification`, `retract`, `assert_node`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `summarize`, `add_nogood`, `_rewrite_dependents`) is tested to leave the index clean — the `TestMutationPathIntegrity` suite.
- `_rewrite_dependents` (`tests/test_dependents_integrity.py:test_api_rewrite_dependents_outlist`) correctly rewrites references appearing in `outlist` as well as `antecedents` when an ID is superseded — same dual-reference rule.

**Diamond graphs and dangling refs.** The storage round-trip test verifies the canonical `A → {B, C} → D` diamond stays consistent across SQLite (`tests/test_dependents_integrity.py:test_round_trip_complex_graph`), and dangling/self-referencing antecedents are tolerated without corrupting the index (`test_dangling_antecedent_reference`, `test_self_referencing_justification`).

## 5. What the observations do **not** show

- The full body of `Network.challenge` / `Network.defend` — so it isn't visible how a *premise* (with no justifications) is knocked OUT by a challenge. Based on the dependents tests it almost certainly synthesizes an SL default justification gated on the challenge node, mirroring `api.py:334`.
- `_cascade_depth` implementation beyond the BFS start (`reasons/api.py:737-745`).
- The actual **CP consistency check** (what "assumptions are consistent" means) — the snippet cuts off before any evaluator that distinguishes `SL` from `CP` during propagation.
- The propagation loop itself (e.g. `propagate`, `retract_node`, `assert_node` bodies at `api.py:475-757`).

These are worth reading directly.

## Topics to Explore

- [file] `reasons/__init__.py` — The `Justification`, `Node`, and `Nogood` dataclasses define the whole type vocabulary — reading the full 60-ish lines makes every other file easier to follow
- [function] `reasons/api.py:defeat_justification` — The canonical "add a defeater to the outlist" primitive that underlies challenges, scopes, and derived GATE suppression; contrasts with `defeat_justification` variants at `api.py:1518` and `api.py:1630`
- [function] `reasons/api.py:_cascade_depth` — The BFS over the dependents index that both computes cascade radius and drives the `depth`/`dependents` fields in `retract`/`assert_node` results
- [function] `reasons/network.py:Network.evaluate` — The actual evaluator that applies the "OR across justifications, AND inside a justification" rule and distinguishes SL from CP; not visible in the supplied snippet but required to fully understand §1–§2
- [general] `cp-consistency-check` — How CP ("assumptions are consistent") is validated during propagation — the snippet only asserts the contract, not the algorithm

## Beliefs

- `in-requires-any-valid-justification` — A node's `truth_value` becomes `"IN"` iff it is a premise (no justifications) or at least one of its justifications is valid (`reasons/__init__.py:30-31`).
- `sl-justification-requires-all-antecedents-in-and-all-outlist-out` — An SL justification is valid iff every antecedent is IN and every outlist node is OUT (`reasons/__init__.py:11-12`, enforced at `reasons/api.py:1590-1592`).
- `dependents-index-maps-both-antecedent-and-outlist-references` — `Y.dependents` contains X whenever X cites Y as an antecedent **or** as an outlist element (`reasons/api.py:2579-2582`, `tests/test_dependents_integrity.py:test_both_antecedent_and_outlist`).
- `challenge-implements-outlist-defeat` — Dialectical challenges are modeled by appending the challenge's node ID to the target's justification `outlist`, so retracting the challenge automatically restores the target (`reasons/api.py:1331-1394`, `tests/test_dialectical.py:test_dismiss_challenge_restores_target`).
- `absent-outlist-node-is-treated-as-out` — A justification may reference an outlist node that does not exist in the network; this is treated as "OUT" and therefore satisfies the outlist condition (`tests/test_outlist.py:test_outlist_node_not_in_network`).
- `any-mode-converts-conjunctive-sl-to-disjunctive` — Passing `any_mode=True` with `sl="a,b"` produces two separate SL justifications, one per antecedent, so the node survives as long as **any** one antecedent holds (`reasons/api.py:306`, `api.py:326`, `api.py:457-460`).
