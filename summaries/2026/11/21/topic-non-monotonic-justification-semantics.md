---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The inlist/outlist justification model (default reasoning via `--unless`) is threaded through `derive.py`, `review.py`, and the dialectic challenge/defend flow; trace it end-to-end.

# Inlist/Outlist Justification Model — End-to-End Trace

## Executive summary

The codebase models Doyle's SL justifications with two slots:

- **inlist** → `Justification.antecedents` — nodes that must be **IN**
- **outlist** → `Justification.outlist` — nodes that must be **OUT**

A node is believed (IN) when *some* justification has all antecedents IN **and** all outlist nodes OUT. That is the "believe X unless Y" default-reasoning primitive. It surfaces through three layers:

1. `reasons/derive.py` — *generates* new outlist-gated justifications via the `### GATE` proposal format.
2. `reasons/review.py` — *audits* those justifications, showing the outlist explicitly and flagging defects.
3. Dialectic challenge/defend (`Network.challenge` / `Network.defend`, tested in `tests/test_dialectical.py`) — *dynamically activates and deactivates counter-evidence*, which behaves like a runtime outlist.

Below is the trace, with a note at the end on what the observations cannot confirm.

> **Caveat on the grep observations.** `grep_inlist_outlist_in_reasons`, `grep_unless_in_reasons`, and `grep_dialectic_challenge_defend` all returned `total_count: 0`, yet the file bodies returned by `read_derive_py` and `read_review_py` **plainly contain `outlist`, `unless`, `GATE`, `Antecedents`, etc.** The greps were almost certainly scoped to the wrong path (e.g., a `reasons/` subdir under a different root, or `.git`-ignored). Treat the file reads and test files as ground truth and disregard the empty-grep counts.

---

## 1. The underlying model (proven by `tests/test_outlist.py`)

The tests are the clearest contract for the semantics that `derive.py` and `review.py` plug into:

- **Construction** — `Justification(type="SL", antecedents=[...], outlist=[...])`. The `antecedents` list is the "inlist"; the `outlist` list has no alias in the constructor.
- **Evaluation** — X is `IN` iff every antecedent is `IN` **and** every outlist node is `OUT`.
  - `test_outlist_valid_when_out` / `test_outlist_invalid_when_in`
  - `test_outlist_with_inlist` (both slots must be satisfied)
  - `test_multiple_outlist_nodes` — "X unless Y **or** Z" means **both** Y and Z must be OUT
  - `test_outlist_node_not_in_network` — a **missing** outlist node is treated as OUT (absence = not believed) → default holds
- **Cascading** — asserting the outlist node retracts the dependent; retracting it restores the dependent (`test_assert_outlist_node_retracts_dependent`, `test_retract_outlist_node_restores_dependent`, `test_outlist_cascade_through_chain`).
- **Multiple justifications** — an alternative justification can keep X IN even when the gated one is defeated (`test_alternate_justification_survives_outlist`).
- **Explanation** — `net.explain("x")` exposes `outlist` when valid and `violated_outlist` when not (`TestOutlistExplain`).
- **Persistence** — outlist survives SQLite round-trip (`TestOutlistPersistence::test_round_trip`).
- **API entry point** — `api.add_node(..., unless="y")` and `api.add_node(..., sl="a", unless="y")`. `test_unless_only_no_sl` proves that `unless=...` with no `sl=` produces a **type: "SL"** node whose justification is *only* the outlist.

This is the layer that `derive.py` writes into and that `review.py` reasons about.

---

## 2. Threading through `reasons/derive.py` (proposal generation)

`derive.py` is the producer of new outlist-gated justifications.

### 2a. The prompt establishes outlist semantics

`DERIVE_PROMPT` (top of file, ~lines 20–130) explicitly teaches the LLM three node kinds, including:

> **Outlist-gated conclusions**: Justified by antecedents UNLESS certain nodes are IN (the conclusion is OUT while the outlist node is IN, and flips IN when it goes OUT)

Task item 3 in the prompt:

> **Connect positive and negative chains** via outlist semantics — where a positive claim should only hold when a negative claim (bug/issue/gap) is OUT

And in `## Rules`:

> For outlist-gated beliefs: the antecedent should be a positive claim, the unless should be a negative claim (bug, gap, issue, fragility)

So the strategy is: **gate a positive conclusion on the absence of a known defect**. This is exactly Doyle's "believe X unless Y is believed."

### 2b. Output format hard-codes GATE as a first-class proposal kind

The `## Output Format` block shows two proposal shapes:

```
### DERIVE <belief-id-in-kebab-case>
<claim>
- Antecedents: ...
- Mode: ALL or ANY
- Label: ...

### GATE <belief-id-in-kebab-case>
<claim>
- Antecedents: ...
- Unless: <comma-separated list of belief IDs that must be OUT>
- Mode: ALL or ANY
- Label: ...
```

The `### GATE` form is the LLM's way of saying "write an SL justification whose `outlist` is the `Unless:` set."

### 2c. `parse_proposals` normalizes GATE → `unless` field

`parse_proposals(response)` (roughly mid-file) uses one regex, `new_pattern`, matching **both** `### DERIVE` and `### GATE`:

```python
new_pattern = re.compile(
    r"### (DERIVE|GATE) (\S+)\n"
    r"(.+?)\n"
    r"- Antecedents: (.+?)\n"
    r"(?:- Unless: (.+?)\n)?"          # ← group 5 = outlist
    r"(?:- Mode: (ALL|ANY)\n)?"
    r"- Label: (.+?)(?:\n|$)",
)
```

Per match it builds:

```python
proposal = {
    "kind": match.group(1).lower(),          # "derive" or "gate"
    "id": match.group(2).strip("`"),
    "text": match.group(3).strip(),
    "antecedents": [...],                    # inlist
    "unless": [u.strip().strip("`")
               for u in match.group(5).split(",")]
              if match.group(5) else [],     # outlist (empty for plain DERIVE)
    "mode": ...,
    "label": ...,
}
```

The legacy `old_pattern` (for `### GATE (outlist): ...` from v0.9) does the same: it detects `GATE` in a nearby header slice and copies `**Unless**:` into `proposal["unless"]`.

**Net effect:** every proposal is normalized to a single dict where `antecedents` is the inlist and `unless` is the outlist (possibly empty). Downstream code doesn't care whether the header was `DERIVE` or `GATE`; it just reads `unless`.

### 2d. `_build_derived_section` renders existing outlist back into the prompt

When `derive.py` re-reads the network for the next round, it surfaces what has already been derived, including outlist:

```python
justs = derived[k]["justifications"]
antes = justs[0].get("antecedents", []) if justs else []
outlist = justs[0].get("outlist", []) if justs else []
...
lines.append(f"- Antecedents: {', '.join(antes)}")
if outlist:
    lines.append(f"- Unless: {', '.join(outlist)}")
```

So the next derivation round sees `- Unless:` lines on prior GATEs, letting the LLM *chain* gates (e.g., derive Z unless W from X unless Y). This is the closure that makes outlist recursive in the prompt.

### 2e. `_get_depth` deliberately ignores outlist

`_get_depth` walks only `j.get("antecedents", [])`. Outlist edges are **not** counted as depth. That's semantically correct: an outlist node being IN *removes* support rather than adding it, so it shouldn't inflate "how deep" a conclusion is. The test `test_outlist_cascade_through_chain` shows cascade behavior still works through antecedents.

### 2f. `build_prompt` tracks outlist in the "referenced" set

In `build_prompt` (below the 500-line mark shown), when computing which nodes have dependents (for `--has-dependents` filtering), it does:

```python
referenced = set()
for v in nodes.values():
    for j in v.get("justifications", []):
        referenced.update(j.get("antecedents", []))
        referenced.update(j.get("outlist", []))
```

So a node that is referenced **only** as an outlist still counts as "has dependents." This matters because retracting it changes the truth value of the gated conclusion — the same property that the TMS uses for cascade.

### 2g. Where proposals become justifications (inferred, not shown)

The 500 lines of `derive.py` we can see stop mid-way through `build_prompt`. The remaining ~197 lines almost certainly contain the code that **persists** proposals — probably looping over `parse_proposals` output and calling `api.add_node(id, text, sl=..., unless=...)` (i.e., the two-argument form tested in `test_add_with_sl_and_unless`). That call is the moment inlist/outlist hits `Network`. This is flagged in the gaps section.

---

## 3. Threading through `reasons/review.py` (semantic audit)

`review.py` is the consumer that validates the justifications `derive.py` produced. The outlist is threaded through both the prompt and the formatter.

### 3a. The prompt establishes the multi-justification rule

`REVIEW_PROMPT` (lines 22–91):

> A belief may have multiple justifications (alternative support paths). **It is valid if ANY justification is sound.** Evaluate each independently.

Combined with outlist semantics, this means: a gated ("believe X unless Y") belief passes review as long as *some* justification is sound — matching `test_alternate_justification_survives_outlist`.

The three axes — valid / sufficient / necessary — include evaluating whether the **outlist justification is necessary** (i.e., is the unless-target actually load-bearing counter-evidence, or could it be dropped?).

### 3b. `format_belief_for_review` surfaces the outlist with full text

This is the core threading point:

```python
for ji, j in enumerate(justs):
    antecedents = j.get("antecedents", [])
    outlist     = j.get("outlist", [])
    label       = j.get("label", "")

    if len(justs) > 1:
        lines.append(f"Justification {ji + 1}/{len(justs)}:")

    if antecedents:
        lines.append("Antecedents:")
        for ant_id in antecedents:
            ant_node = nodes.get(ant_id)
            ...

    if outlist:
        lines.append("Unless (must be OUT):")
        for out_id in outlist:
            out_node = nodes.get(out_id)
            if out_node:
                lines.append(f"- {out_id}: {out_node.get('text', '')}")
            else:
                lines.append(f"- {out_id}: (not found in network)")
```

Two things to note:

1. **Missing outlist nodes are tolerated** and flagged as `(not found in network)`. This is the same discrimination as `test_outlist_node_not_in_network` — the reviewer sees a node that will be treated as *absent = OUT*, so the default holds. The reviewer can then judge whether that is acceptable for this particular gated conclusion.
2. **Each justification is presented independently** with its own inlist/outlist pair, so the LLM can apply the "any justification survives" rule from the prompt.

### 3c. Defeat reason taxonomy maps to outlist abuse

`DEFEAT_REASON_TYPES` and `CLASSIFY_DEFEAT_REASON_PROMPT` include categories that are exactly the failure modes of default reasoning:

- `over-generalizes` — "believe X unless Y" where Y is too narrowly scoped to cover the real exceptions
- `scope-mismatch` — outlist covers a narrower scope than the conclusion claims
- `missing-bridge` — the unless-node is unrelated to the conclusion it's supposed to gate
- `unsupported-conjunct` — a conjunction in the positive claim has no single antecedent establishing it

`classify_defeat_reason(defeater_text, defeated_text, model, timeout)` (lines ~130–160) is a second LLM pass that tags a defeat verdict with one of these, so downstream consumers can graph "what kind of reasoning failure this was."

### 3d. `review_beliefs` selection includes outlist-only justifications

```python
if belief_ids is None:
    belief_ids = [
        nid for nid, node in sorted(nodes.items())
        if node.get("truth_value") == "IN"
        and node.get("justifications")
        and len(node["justifications"]) > 0
    ]
```

Because `api.add_node(..., unless=...)` with no `sl=` produces a node with an **SL justification containing only an outlist** (`test_unless_only_no_sl` confirms `result["type"] == "SL"`), that node **does** satisfy `justifications and len > 0` and gets reviewed. So default (unless-only) premises are audited alongside multi-antecedent derivations.

### 3e. The output schema — where review hands off to dialectic

`parse_review_response` returns:

```python
{
    "id": ...,
    "valid": ...,
    "sufficient": ...,
    "necessary": ...,
    "unnecessary_antecedents": [...],
    "scope_findings": [ {antecedent, establishes, does_not_establish} ],
    "missing_property": ...,
    "defeat_reason_type": ...,
}
```

`scope_findings` + `missing_property` are the structured payload that, in a later step, can be turned into **a challenge node** (dialectic). The observation set does not show that translation, but the schema is clearly shaped to feed it — each `scope_finding` is essentially "here is what this inlist element fails to establish," i.e., material for an `challenge-<target>` node.

---

## 4. Threading through dialectic challenge/defend (`tests/test_dialectical.py`)

The dialectic flow is the **runtime** expression of the same inlist/outlist primitive, but with argumentative nodes instead of a static outlist. The implementation isn't in the observation dump, but the tests pin down the observable behavior.

### 4a. `challenge` = "assert a counter-belief that retracts the target"

`TestChallenge`:

- `net.challenge("a", "A is wrong because X")` →
  - Creates node `challenge-a`, `truth_value == IN`, `metadata["challenge_target"] == "a"`
  - Back-reference `metadata["challenges"] == ["challenge-a"]` on the target
  - Target `a` flips to `OUT`
  - Result dict: `{"challenge_id": "challenge-a", "changed": [...] }`
- Cascades normally: `test_challenge_cascades` — challenging A retracts B which depends on A.
- Multiple challenges get suffixed ids (`challenge-a`, `challenge-a-2`) and are appended to `metadata["challenges"]`.
- `retract("challenge-a")` → target restored (`test_dismiss_challenge_restores_target`).

The mechanical equivalent of a challenge is: **the target's existing justification is treated as outlist-gated on `challenge-<target>` being OUT**. When the challenge is asserted (IN), the target's justification is violated → target goes OUT. When the challenge is retracted (OUT), the default holds again → target back IN. This is exactly `test_outlist_invalid_when_in` inverted — the same evaluator.

### 4b. `defend` = "assert a counter-counter-belief"

`TestDefend`:

- `net.defend("a", "challenge-a", "A is right because Y")` →
  - Creates `defense-challenge-a` with `truth_value == IN`, `metadata["defense_target"] == "challenge-a"`, `metadata["defends"] == "a"`
  - `challenge-a` flips to `OUT`
  - Because the challenge is now OUT (as an outlist node for A is OUT), A's original support is re-enabled → A flips to `IN`
  - Cascades restoration to dependents (`test_defend_cascades_restoration`)

So `defend` doesn't touch A directly. It **retracts the challenge**, and the TMS re-evaluates A's justification (outlist-now-satisfied) → IN. It's a clean outlist relaxation.

### 4c. Multi-level chain = nested outlists

`TestDialecticalChain::test_challenge_defense`:

1. `challenge("a")` → A OUT
2. `defend("a", "challenge-a")` → challenge OUT → A IN
3. `challenge("defense-challenge-a")` → defense OUT → **challenge-a back IN** → **A OUT**

Each level is just another outlist relationship. The evaluator handles arbitrary depth for free because each node's justification only sees its immediate antecedents/outlists. `test_three_level_chain` proves 4 levels work.

### 4d. Persistence = truth values are computed, not stored

`TestDialecticalPersistence::test_round_trip` reloads the DB, then `retract("defense-challenge-a")` and observes the challenge comes back IN and A flips OUT. Since truth values are recomputed from justifications + metadata (`challenge_target` / `defense_target` / back-references), the outlist is **fully reconstructed** on load. This matches `TestOutlistPersistence::test_round_trip`.

### 4e. The API surface for dialectic

`TestDialecticalAPI`:
- `api.challenge("a", "A is wrong", db_path=db)` returns `{"challenge_id": "challenge-a", "changed": [..., "a"]}`
- `api.defend("a", "challenge-a", "A is right", db_path=db)` returns `{"defense_id": "defense-challenge-a", ...}`
- `api.list_nodes(challenged=True, db_path=db)` filters to nodes with non-empty `metadata["challenges"]`

Those APIs are the ones `review.py`'s defeat verdicts would call to instantiate a challenge from a failed review — the wiring is not visible in the observations, but the shape is clear.

---

## 5. End-to-end narrative (one loop)

```
                    ┌───────────────────────────────────────────────┐
                    │  Network: Justification(type="SL",           │
                    │      antecedents=[...], outlist=[...])       │
                    │  → IN iff all ante IN and all outlist OUT    │
                    └───────────────────────────────────────────────┘
                            ▲                             │
               api.add_node │                             │ _build_derived_section
               (unless=...) │                             │ reads j.outlist → "- Unless:"
                            │                             ▼
                    ┌───────┴───────┐           ┌───────────────────────┐
                    │  derive.py    │           │  derive.py builds new │
                    │  parse_       │──────────▶│  prompt from network  │
                    │  proposals    │  GATE     │  (round N+1)          │
                    │  ⟶ "unless"   │           └───────────────────────┘
                    └───────────────┘
                            │ (persists via api.add_node)
                            ▼
                    ┌───────────────┐
                    │  review.py    │  format_belief_for_review →
                    │  audits new   │  "Unless (must be OUT): ..."
                    │  GATEs        │
                    └───────┬───────┘
                            │ invalid / insufficient
                            ▼
                    ┌───────────────┐
                    │  dialectic    │  challenge(target, reason)
                    │  (Network.    │  → creates challenge-<target> IN
                    │  challenge/   │  → target's justification's outlist
                    │  defend)      │     effectively gains a new entry
                    └───────────────┘
                            │ user (or reviewer) disagrees
                            ▼
                        defend(target, challenge_id, reason)
                            → retracts challenge
                            → outlist satisfied again
                            → target restored
```

Key invariants that make this loop tidy:

1. **derive.py never does truth evaluation.** It only writes justifications with an outlist.
2. **review.py never writes justifications.** It only reads them and emits structured verdicts (`scope_findings`, `defeat_reason_type`).
3. **dialectic never bypasses the SL layer.** `challenge` and `defend` create ordinary nodes whose IN/OUT status flows through the same `antecedents`/`outlist` evaluator as everything else — the recursion through `test_three_level_chain` is evidence there is no special-casing.
4. **Missing outlist nodes are safe** (`test_outlist_node_not_in_network`, `format_belief_for_review`'s `(not found in network)` fallback). So a GATE can reference a hypothesis that hasn't been added yet; the default holds until someone `add_node`s it.

---

## 6. What is missing from the observations

The trace above is grounded in what the observation bundle contains, but three seams are not shown and would need a fresh read:

1. **The final ~197 lines of `reasons/derive.py`** (`lines 500–697`). This is where proposals are persisted — presumably a `process_proposals` / `apply_derivations` loop that calls `api.add_node(id, text, sl=",".join(antecedents), unless=",".join(unless))`. Without it we cannot confirm the exact argument shape used.
2. **The implementation of `Justification` evaluation and `Network.challenge` / `Network.defend`.** The tests pin down the *behavior*, but not the code path. Specifically: does `challenge` add an outlist entry to each of the target's justifications, or does it install a fresh justification "IN unless challenge-x"? The tests can't distinguish these.
3. **The review → dialectic seam.** `parse_review_response` emits `scope_findings` / `missing_property` / `defeat_reason_type`, but nothing in the observations calls `api.challenge(...)` with those. Likely in a higher-level orchestrator (`reasons/cli.py` or `reasons/pipeline.py`), or in the unseen tail of `derive.py`.

Additionally, the three empty greps contradict the actual file contents and should be re-run against the real package root (`find reasons -name '*.py' -exec grep -n "outlist\|unless" {} +`) to be trusted.

---

## Topics to Explore

- [file] `reasons/derive.py` — Read the tail (lines ~500–697) to see the function that turns `parse_proposals` output into `api.add_node(sl=..., unless=...)` calls; this is the concrete inlist/outlist persistence point.
- [function] `reasons/network.py:Network.challenge` — Determine whether `challenge` mutates each target justification's outlist vs. installing a new "IN unless challenge-x" justification; the dialectic tests pass either way.
- [function] `reasons/api.py:add_node` — Trace how `sl="a"` becomes `Justification(antecedents=["a"])` and how `unless="y"` becomes `outlist=["y"]`, including the `test_unless_only_no_sl` case that yields `type: "SL"` with an empty antecedent list.
- [general] `review-to-dialectic-wiring` — Find the orchestrator that converts `review.py`'s `scope_findings` / `missing_property` into a `challenge(...)` call; the schema strongly implies this exists but no observation shows it.
- [file] `reasons/storage.py` — Confirm that `justifications.outlist` (a JSON column?) and the `metadata["challenges"]` / `metadata["defends"]` back-references round-trip through SQLite, which both `TestOutlistPersistence` and `TestDialecticalPersistence` assert.

## Beliefs

- `parse-proposals-emits-unless-field` — `parse_proposals` in `reasons/derive.py` captures the `- Unless:` line into `proposal["unless"]` for both `### GATE` and legacy formats, and defaults to `[]` when absent.
- `build-derived-section-renders-outlist` — `_build_derived_section` emits a `- Unless: ...` line from `justs[0]["outlist"]` whenever the first justification has a non-empty outlist.
- `review-format-includes-outlist-context` — `format_belief_for_review` renders an `Unless (must be OUT):` block per justification, and falls back to `(not found in network)` for outlist IDs missing from `nodes`.
- `outlist-absent-node-treated-as-out` — A missing outlist node is treated as OUT, so a default justification with an absent `unless` target evaluates to IN (`test_outlist_node_not_in_network`).
- `outlist-only-api-node-is-sl-typed` — `api.add_node(..., unless="y")` with no `sl=` produces a node whose `type` is `"SL"` with a single justification whose only slot is the outlist (`test_unless_only_no_sl`).
- `challenge-creates-counter-node` — `Network.challenge(target, reason)` creates a `challenge-<target>` node with `truth_value == IN` and retracts the target, cascading to dependents.
- `defend-retracts-challenge-to-restore-target` — `Network.defend(target, challenge_id, reason)` creates a `defense-<challenge_id>` node with `truth_value == IN` and flips `challenge_id` OUT, which re-satisfies the target's outlist and restores it to IN.
