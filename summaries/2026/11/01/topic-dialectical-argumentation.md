---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Topic: The challenge/defend mechanism: how non-monotonic outlists enable formal belief challenging and defense. Spans `network.py`, `repair.py`, `contradictions.py`

# The Challenge/Defend Mechanism: Non-Monotonic Outlists and Belief Challenging

## The Core Idea

This system implements a **Truth Maintenance System (TMS)** in the tradition of Doyle's Non-Monotonic Logic. The central insight is that beliefs aren't just asserted — they carry *defeasible* justifications. A belief can be IN (held true), but that truth is *conditional on the absence of counter-evidence*. The three modules you're looking at form a pipeline: **discover conflicts → formally retract → attempt repair**.

---

## 1. The Formal Mechanism: Outlists in `network.py`

The logical backbone lives in the `Network` class and the `Justification` data structure. A justification has two lists:

- **Inlist (antecedents):** nodes that *must* be IN for the justification to hold.
- **Outlist:** nodes that *must* be OUT for the justification to hold.

This is the "believe X *unless* Y" pattern, made explicit in the test file header (`tests/test_outlist.py`, line 1):

> Doyle's key innovation: SL justifications have both an inlist (must be IN) and an outlist (must be OUT). "Believe X unless Y is believed."

The tests in `TestOutlistBasic` (lines 14–78) demonstrate the full truth table:

| Inlist status | Outlist status | Result |
|---|---|---|
| All IN | All OUT | **IN** (justification holds) |
| Any OUT | — | **OUT** (antecedent missing) |
| All IN | Any IN | **OUT** (defeated) |
| — | Node absent | **IN** (absent ≡ not believed) |

The critical non-monotonic behavior is in `TestOutlistCascading` (lines 80+):

```python
def test_assert_outlist_node_retracts_dependent(self):
    """X unless Y. Assert Y → X goes OUT."""
    changed = net.assert_node("y")
    assert net.nodes["x"].truth_value == "OUT"
    assert "x" in changed

def test_retract_outlist_node_restores_dependent(self):
    """X unless Y. Y is IN (X is OUT). Retract Y → X restored."""
    changed = net.retract("y")
    assert net.nodes["x"].truth_value == "IN"
```

This is what makes the system **non-monotonic**: asserting a defeater *removes* a belief, and retracting the defeater *restores* it. The `Network.assert_node` and `Network.retract` methods trigger cascading propagation through the outlist dependencies. A single assertion can knock out an entire chain of dependents.

> **Note:** The search for "outlist" / "defeater" / "challenge" / "defend" in `network.py` returned zero matches. The actual implementation of the propagation logic (how `assert_node` walks the outlist dependency graph) is not visible in the observations. The tests confirm the *behavioral contract*, but the internal cascade algorithm in `network.py` is a gap in what we can see here.

---

## 2. Challenge Discovery: `contradictions.py`

Before you can formally retract a belief via its outlist, you need to *identify* which beliefs conflict. That's the job of `detect_contradictions` in `reasons/contradictions.py` (line 122+).

The function:
1. Collects all currently **IN** beliefs (line 137–141).
2. Batches them (50 per call, `CONTRADICTION_BATCH_SIZE` at line 14).
3. Sends each batch to an LLM with a structured prompt asking it to find **NOGOODs** — sets of ≥2 beliefs that "cannot all be true simultaneously."

The four contradiction types the prompt enforces (lines 24–33):
- **Direct negation:** A says X, B says ¬X
- **Incompatible properties:** A says X has P, B says X has Q, P ⊥ Q
- **Scope conflict:** A says "all X are Y," B provides a counterexample
- **Quantitative mismatch:** A implies a bound incompatible with B

Each NOGOOD gets an `id`, a list of `claims` (belief IDs), an `analysis`, and a `severity` (High/Medium/Low). The parser (`parse_contradiction_response`, line 72) extracts these via regex and filters against valid node IDs.

**How this connects to the outlist:** The NOGOOD detection is the *trigger*. Once you've identified that beliefs A and B can't both be IN, you need a mechanism to retract one of them. That mechanism is the outlist — you'd wire B into A's outlist (or vice versa), so that asserting B formally defeats A through the TMS propagation.

---

## 3. Defense: `repair.py`

Not every challenged belief should be blindly retracted. `repair.py` implements a **defense pipeline** that attempts to save a belief before full retraction. The module docstring (line 4–9) names three patterns:

1. **search-and-link** — the belief is sound but missing an antecedent; find and wire it in.
2. **soften** — the belief overstates the evidence; weaken the text.
3. **abandon** — the dependency chain is too broken; retract.

(Plus a fourth, **research**, added later for cases where the belief is plausible but unconfirmable.)

The flow in `repair_beliefs` (the orchestrator, not fully shown but referenced at line 9):

1. **Triage** (`TRIAGE_PROMPT`, line 72): An LLM looks at the invalid belief + review comment + dependency info and picks a pattern.
2. **Extract** (`EXTRACT_PROMPT`, line 12): For `search-and_link`, identify the "smuggled claim" — the specific factual assertion the conclusion relies on but the antecedents don't provide.
3. **Match** (`MATCH_PROMPT`, line 28): Search existing premises in the network for one that supports the smuggled claim.
4. **Execute**: Either link the found premise, rewrite the belief text (soften), or call `retract` (abandon).

**How this connects to the outlist:** If repair succeeds (search-and-link finds a missing antecedent, or soften rewords the belief), the belief stays IN. If repair fails (abandon), the belief is retracted — and any beliefs that had it in their *inlist* cascade OUT. If the belief had other nodes in its *outlist*, those are unaffected. The outlist mechanism is what makes the final "abandon" step a clean, propagation-safe retraction rather than a manual deletion.

---

## 4. Defeat Reason Classification: `review.py`

The test file `tests/test_classify_defeaters.py` reveals a metadata layer on top of the mechanism. When a justification is defeated via `api.defeat_justification(node_id, just_index, reason_text, defeater_type)`, the defeater node records *why* it defeated:

- `unsupported-conjunct` — a conjunct in the inlist isn't actually established
- `over-generalizes` — the conclusion extends beyond what the antecedents support
- `scope-mismatch` — the quantifier scope is wrong
- `migrated-retraction` — the belief was retracted during a migration

The function `classify_defeat_reason` (in `reasons/review.py`, called from `reasons/llm.py`) asks an LLM to pick the correct category from `DEFEAT_REASON_TYPES`. `api.classify_defeat_reason_types` (line 118 of the test) runs this across all unclassified defeaters, with a `dry_run` flag to preview before writing `defeat_reason_type` into the node's metadata.

This gives you a **formal taxonomy of why a challenge occurred**, which is essential for auditing and for the repair module to choose the right defense pattern.

---

## Putting It All Together

```
                    ┌─────────────────────┐
                    │  contradictions.py  │  (CHALLENGE DISCOVERY)
                    │  detect_contradictions│
                    │  → finds NOGOOD sets  │
                    └────────┬────────────┘
                             │  "Beliefs A and B can't both be IN"
                             ▼
                    ┌─────────────────────┐
                    │     network.py      │  (FORMAL REJECTION)
                    │  Justification      │
                    │  inlist + outlist   │
                    │  → assert/retract   │
                    │    cascade          │
                    └────────┬────────────┘
                             │  "Belief X is now OUT"
                             ▼
                    ┌─────────────────────┐
                    │     repair.py       │  (DEFENSE)
                    │  repair_beliefs     │
                    │  → triage: link /   │
                    │    soften / abandon │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        "Link missing"  "Rewrite"    "Retract"
        antecedent      the text     (final OUT)
              │              │              │
              ▼              ▼              ▼
        X stays IN     X stays IN     X is OUT,
        (repaired)     (weakened)     cascade propagates
```

The outlist is what makes this cycle **reversible and composable**. You can assert a defeater, watch the cascade, then retract the defeater and watch the belief restore — without any manual bookkeeping. The contradiction detector finds *what* to challenge, the repair module decides *whether* to defend, and the network's outlist mechanism executes the *how*.

---

## What's Missing from the Observations

- **`network.py` internals:** The actual cascade/propagation algorithm in `Network.assert_node` and `Network.retract` isn't visible. We know the behavioral contract from tests, but not whether it's BFS, DFS, or iterative fixpoint.
- **The wiring between `contradictions.py` and `network.py`:** Who calls `defeat_justification` after a NOGOOD is found? Is there an API function that takes a NOGOOD and automatically wires the outlist? This orchestration glue isn't in the observations.
- **`repair.py` full source:** We see the prompts and the first 150 lines of parsing, but not the actual `repair_beliefs` orchestrator loop or how it calls `Network` methods.
- **`Justification` dataclass definition:** Referenced in tests but not shown. Its `type="SL"` field (Semi-Monotonic Logic?) and the `outlist` field structure would clarify the formal semantics.

---

## Topics to Explore

- [file] `reasons/network.py` — The cascade/propagation algorithm in `assert_node` and `retract`; how outlist dependencies are walked and how the fixpoint is computed
- [function] `reasons/repair.py:repair_beliefs` — The orchestrator loop that triages, dispatches to a pattern, and calls back into `Network` for link/soften/abandon
- [function] `reasons/api.py:defeat_justification` — The public API that wires a defeater into a justification's outlist; the bridge between the contradiction/repair layers and the TMS
- [file] `reasons/review.py` — `classify_defeat_reason` and `DEFEAT_REASON_TYPES`; how the defeat taxonomy maps to repair pattern selection
- [general] `nogood-to-outlist-wiring` — The orchestration path from `detect_contradictions` output to actual `defeat_justification` calls; is there an automatic pipeline or is it manual?

---

## Beliefs

- `outlist-presence-defeats-belief` — In an SL justification, any outlist node being IN causes the justified node to be OUT, regardless of inlist status
- `outlist-retraction-restores-belief` — Retracting an outlist node that was causing a defeat restores the dependent node to IN, propagating upward through the dependency chain
- `nogood-minimum-two-claims` — A NOGOOD from `contradictions.py` must contain at least 2 belief IDs to be valid; single-claim "contradictions" are filtered out by `parse_contradiction_response`
- `repair-patterns-exhaustive` — The `repair.py` triage classifies every invalid belief into exactly one of four patterns (`search_and_link`, `soften`, `abandon`, `research`); there is no fifth "keep as-is" option
- `absent-outlist-node-treated-as-out` — An outlist node that doesn't exist in the network is treated as OUT (not believed), so the justification remains valid; this is confirmed by `test_outlist_node_not_in_network`
