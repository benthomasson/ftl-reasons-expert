---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: The classic Truth Maintenance System paper that this algorithm (nogood → backtrack to premises → retract) implements; useful for reconciling terminology like "premise", "justification", "entrenchment".

# The Paper: Doyle (1979), *A Truth Maintenance System*

The codebase pins its lineage on the first line of the package:

> `reasons/__init__.py:1` — `"""Reason Maintenance System — data model based on Doyle (1979)."""`

That is the canonical reference:

> **Jon Doyle. "A Truth Maintenance System." *Artificial Intelligence* 12(3):231–272, 1979.**

Essentially every load-bearing term in this repo — IN/OUT labels, justifications, premises, outlists for non-monotonic reasoning, dependency-directed backtracking to retract a premise — is introduced in that paper. Almost every other term ("nogood", "entrenchment", SL/CP) is a continuation of it in another thread of the TMS literature, and it is worth knowing which is which so you don't misattribute.

## How this code implements Doyle

### IN / OUT labels and the "premise" primitive

`reasons/__init__.py:30-31`:

```python
# A node is IN if ANY of its justifications is valid.
# A premise has no justifications and is IN by default.
```

That is a direct restatement of Doyle's node-label machinery. A **premise** is exactly "a node with an empty justification list"; the IN/OUT label is derived from justification validity, not stored as truth.

The same definition shows up operationally in `reasons/repair.py`:

```python
def _compute_depth(node_id, nodes, memo=None):
    ...
    node = nodes.get(node_id)
    if not node or not node.get("justifications"):
        memo[node_id] = 0
        return 0
```

A node with no justification is depth-0 — i.e. a premise/assumption. Everything else is `max(antecedent depth) + 1`. This is the "assumption" notion Doyle (1979) uses as the base case of dependency-directed backtracking: the culprits for a contradiction are always assumptions somewhere at the leaves of the derivation.

### Justifications: SL, outlist, and the "unless" clause

`reasons/api.py:300-306` exposes the classic Doyle signature:

```python
# sl: Comma-separated antecedent IDs for SL justification
# cp: Comma-separated antecedent IDs for CP justification
# unless: Comma-separated outlist IDs (must be OUT for justification to hold)
```

and `reasons/api.py:321-334` builds them:

- `Justification(type="SL", antecedents=[...], outlist=[...], label=...)`
- `Justification(type="CP", antecedents=[...], outlist=[...], label=...)`
- `Justification(type="SL", antecedents=[], outlist=[...], label=...)` — "premise that holds unless something is believed"

That last case (`reasons/api.py:333-334`) is Doyle's **non-monotonic justification**: an empty in-list with a non-empty out-list is a defeasible premise. `reasons/__init__.py:16-17` spells out its semantics:

> "…believed." If Y comes IN, the justification becomes invalid and X may go OUT (if no other justification supports it).

That is Doyle's `(SL () (Y))` — believe X unless Y is IN. The **SL** name (Support List) is Doyle's; the **CP** justification type is the standardized later addition in McAllester (1990, *Truth Maintenance*, AAAI) and Forbus & de Kleer (1993, *Building Problem Solvers*), which is the canonical "as-implemented" reference for this exact data model. If you only read one book about the algorithm *this repo implements*, read Forbus & de Kleer, ch. 3–4 — it reconstructs Doyle's algorithm line-for-line with the SL/CP vocabulary the API uses.

### Nogood → trace → retract (dependency-directed backtracking)

The paper's central procedure is what the tests call *dependency-directed backtracking* (`tests/test_backtracking.py` module docstring):

> "When a contradiction (nogood) is discovered, the system traces backward through the justification graph to find the premises responsible, then retracts the premise with minimal disruption instead of an arbitrary node."

The three-step cycle in the code:

1. **Detect the nogood.** `reasons/contradictions.py` — `CONTRADICTION_PROMPT` asks the model to emit `### NOGOOD <id>` blocks with `Claims:` (a set of node IDs). `parse_contradiction_response` (same file) extracts them into `{"id", "claims", "analysis", "severity"}` dicts. IDs are content-addressed by sorted node IDs — see `tests/test_nogood_id.py:TestContentBasedIds`: `add_nogood(["b","a"])` → `"nogood-a-b"`, with `-2` suffixing on duplicates.

2. **Trace to premises.** `Network.trace_assumptions(node_id)` walks back through justifications and returns the list of premise IDs. The tests are the clearest spec:
   - `tests/test_backtracking.py:TestTraceAssumptions::test_chain_traces_to_root` — `A → B → C` returns `["a"]`.
   - `::test_diamond_traces_to_root` — diamond shape still collapses to `["a"]`.
   - `::test_multiple_premises` — `A, B → C` returns `{"a","b"}`.

   This is Doyle's "reason for a node" walk; the API wrapper is `api.trace_assumptions` (`tests/test_backtracking.py:TestBacktrackingAPI::test_trace_api`), returning `{"node_id", "premises"}`.

3. **Retract the least-entrenched culprit.** `Network.find_culprits(claim_ids)` returns one entry per leaf premise that appears in the nogood's derivation, each with a `would_resolve` list of which nogood members it covers (`tests/test_backtracking.py:TestFindCulprits::test_would_resolve_field`). Culprits are sorted by number of dependents ascending so retracting the first causes the smallest cascade (`::test_sorted_by_dependents`). `Network.add_nogood(...)` then does the retraction and records a `backtrack` log entry with the culprit (`::test_backtracking_logs_culprit`), and `api.add_nogood` surfaces `{"backtracked_to": ...}` to callers (`::test_nogood_reports_backtrack`).

Two behavioral details in that directory worth noting, because they are textbook Doyle:

- **The retraction is a premise, never a derived node.** `tests/test_backtracking.py::TestBacktrackingInNogood::test_retracts_premise_not_derived` — when `B` and `C` (derived from `A`) contradict, the TMS retracts `A` (or `B`), never `C` directly. Derived nodes lose support only via cascade.
- **A shared premise resolves both sides.** `::test_shared_premise_resolves_both` — `A → B`, `A → C`, nogood `(B, C)` — retracting `A` turns off both `B` and `C`. This is Doyle's *minimal, dependency-directed* retraction: pick the assumption common to both horns of the contradiction rather than the "closest" node to the contradiction.

### Entrenchment (the tie-breaker)

Where multiple candidate culprits would each resolve the nogood, `Network._entrenchment(node_id)` decides which belief survives. `tests/test_backtracking.py:TestEntrenchment` is the spec:

- `::test_evidence_premise_more_entrenched_than_derived` — a premise with a `source` outranks a bare premise.
- `::test_sourced_premise_over_unsourced` — source presence is an entrenchment signal.
- `::test_beliefs_type_affects_entrenchment` — `OBSERVATION > DERIVED` via `metadata["beliefs_type"]`.
- `::test_more_dependents_more_entrenched` — fan-out is a signal (don't kill a premise half the network depends on).

That list is closer to the **AGM "epistemic entrenchment"** notion (Gärdenfors & Makinson 1988, *Relations Between the AGM Theory and Epistemic Entrenchment*) than to anything in Doyle 1979, and the `_entrenchment` name is an intentional gesture at that literature. Doyle 1979 uses a related "well-foundedness" / "priority" idea for the same purpose (breaking ties during DDB); the AGM reframing is what gives you the tidy "source > no source, observation > derived, depended-upon > orphan" ranking in the tests. So: **Doyle is the algorithm; AGM is the entrenchment scoring.**

## Terminology reconciliation table

| Term in this repo | Doyle (1979) meaning | Notes |
|---|---|---|
| **premise** | Node with no justifications, IN by default (`reasons/__init__.py:31`) | Called "assumptions" in DDB walk; `_compute_depth` treats them as depth 0. |
| **justification** | `(SL (in-list) (out-list))` — IN iff all in-list IN and all out-list OUT | `Justification` dataclass in `reasons/__init__.py:36`. `api.py` exposes `sl`, `cp`, `unless`. |
| **SL** | Support List justification | The core type. |
| **CP** | Conditional-Proof justification | Not in Doyle 1979; standardized in McAllester (1990) / Forbus & de Kleer (1993). |
| **outlist / "unless"** | Doyle's non-monotonic justification | `reasons/__init__.py:16-17`; API kwarg `unless` (`api.py:302`). |
| **nogood** | (Doyle: set of justifications in conflict; term itself is ATMS-era) | The word *nogood* is most associated with de Kleer 1986 *An Assumption-Based TMS*; this repo is a single-label JTMS that adopts the ATMS term for "set of nodes that cannot all be IN." |
| **dependency-directed backtracking** | Doyle 1979 §6 | `trace_assumptions` → `find_culprits` → `add_nogood` → `retract`. |
| **entrenchment** | Doyle 1979 uses term loosely; AGM formalizes it | `_entrenchment` in `Network`; ranking rules tested in `TestEntrenchment`. |
| **retract** | Set a node OUT, cascade via justify-node | SKILL.md:73-76. |
| **assert / restore** | Re-derive; dependents recomputed | SKILL.md:79. |

## On repair_* and the LLM layer

Note that `reasons/repair.py` and `reasons/repair_premises.py` are **not** part of Doyle. They sit above the TMS as a triage layer: when a *premise's text* is wrong, the system uses an LLM to decide **rewrite vs. retract** (`REPAIR_PREMISES_PROMPT` in `reasons/repair_premises.py`) or **search-and-link vs. soften vs. abandon vs. research** (`TRIAGE_PROMPT` in `reasons/repair.py`). The closed-form Doyle algorithms are in `reasons/network.py` (not in the observations above — only its tests are); the LLM code is an application layer that feeds corrected premises back into that algorithm.

If you find a doc that calls the whole thing "a TMS", read it as "a Doyle JTMS with an LLM-driven *premise repair* layer on top; nogoods and DDB are pure Doyle; entrenchment is AGM-inflected."

## Topics to Explore

- [file] `reasons/network.py` — Defines `trace_assumptions`, `find_culprits`, `_entrenchment`, and `add_nogood` — the actual Doyle DDB implementation, which the tests exercise but which wasn't included in the observations.
- [function] `reasons/network.py:Network.add_nogood` — Where the "nogood → trace → retract culprit" cycle lands; the `{"backtracked_to": …}` contract in `api.add_nogood` is enforced here.
- [function] `reasons/network.py:Network.justify_node` (or equivalent) — The IN/OUT propagation loop that Doyle's paper is really about; needed to confirm the SL/outlist semantics line up exactly with `(SL (in) (out))`.
- [file] `reasons/contradictions.py` — Also useful to compare `detect_contradictions` vs `detect_contradictions_semantic`; the latter clusters first, which affects whether a nogood is *minimal* (Doyle only cares about minimal culprits; non-minimal nogoods can retract more than necessary).
- [general] `Doyle 1979 vs de Kleer 1986 vs AGM` — A short reading list to nail terminology: Doyle 1979 for the algorithm, de Kleer 1986 for the noun *nogood*, Gärdenfors–Makinson 1988 or Doyle 1983 *The Ins and Outs of Reason Maintenance* for *entrenchment*.

## Beliefs

- `doyle-1979-is-the-algorithm-reference` — `reasons/__init__.py:1` explicitly attributes the data model to Doyle (1979), and `justifications`, `premise`, `retract`, `outlist` correspond to that paper's primitives.
- `premise-is-defined-as-node-with-empty-justification-list` — `reasons/__init__.py:31` and `reasons/repair.py:_compute_depth` both define a premise as a node whose `justifications` list is empty, with depth 0.
- `justification-requires-outlist-members-to-be-out` — The outlist / `unless` kwarg (`reasons/api.py:302`) must be OUT for a justification to hold; this is non-monotonic and documented at `reasons/__init__.py:16-17`.
- `nogood-retraction-requires-tracing-to-premises-first` — `Network.add_nogood` always calls `trace_assumptions` and `find_culprits` before retracting; `tests/test_backtracking.py::TestBacktrackingInNogood::test_retracts_premise_not_derived` asserts that only premise nodes (never the derived nogood members themselves) are direct retraction targets.
- `entrenchment-prevents-retracting-evidence-over-speculation` — `tests/test_backtracking.py::TestEntrenchment::test_nogood_retracts_speculation_not_evidence` encodes the invariant that a source-backed `OBSERVATION` premise must survive a contradiction against a `DERIVED` node.
- `nogood-ids-are-content-addressed` — `tests/test_nogood_id.py:TestContentBasedIds` guarantees `add_nogood(["b","a"])` yields `"nogood-a-b"` (sorted, joined, `-N` suffixed on duplicate), which is what makes `parse_contradiction_response` safe to trust against `valid_ids`.
- `llm-repair-layer-is-above-doyle` — `reasons/repair.py` and `reasons/repair_premises.py` produce corrected *premise text* / *new antecedents* but do not themselves implement justification propagation or DDB; those remain in `reasons/network.py`.
