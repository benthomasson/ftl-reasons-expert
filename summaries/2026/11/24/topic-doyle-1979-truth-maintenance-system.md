---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The source paper; read §1–3 for SL/CP justifications and §6 for the dialectical framing cited in `tests/test_dialectical.py`.

# The source paper — Doyle (1979), and how the codebase uses §§1–3 and §6

## Important caveat first

**The paper text itself is not in the observations.** What we have is the *codebase's* references to it — the README, the test docstring, and the SL/CP `Justification` dataclass. So this explanation reconstructs what the code claims the paper says, and points out where the paper itself would need to be read to confirm anything.

The single citation of the paper is in `README.md` line 199:

> Doyle, J. (1979). A Truth Maintenance System. *Artificial Intelligence*, 12(3), 231–272.

Everything else (§ numbering, "Doyle Section 6") is imposed by the codebase on top of that reference.

---

## §1–3 — SL/CP justifications as the code understands them

The data model and invariants only make sense if you read them as a direct transcription of Doyle's justification machinery. These are the load-bearing facts:

- **`Justification.type` is a string, either `"SL"` or `"CP"`** — see `reasons/__init__.py:19` (`type: str  # "SL" or "CP"`) and `CLAUDE.md:42`. Everything downstream branches on this string.
- **SL (Support List)** — `reasons/__init__.py:11` and `README.md:9`: a node is IN iff **all** antecedents are IN. This is the conjunctive/default justification type; a premise is just a node with no justifications, so it is IN by default (`CLAUDE.md:8`).
- **CP (Conditional Proof)** — `reasons/__init__.py:13`: a node is IN iff the assumptions are consistent. Shaped at build time by `api.py:331`, `api.py:444` (`jtype = "CP"`), and documented in `README.md:167` (`--cp a,b`).
- **Outlist / non-monotonic layer** — `README.md:15` and `CLAUDE.md:11`: a justification is valid iff all `inlist` (antecedents) are IN **and** all `outlist` nodes are OUT. This is the "believe X unless Y" default rule, exposed in the API via `--unless` (`CLAUDE.md:57`) and `outlist` parameters in `api.py:326–334`.
- **Multiple justifications** — `README.md:11`: a node is IN if *any* of its justifications is valid. This is a Bellman/AND-of-ORs encoding: each justification is an AND of inlists minus outlists; the node is an OR of justifications.
- **`any_mode`** — `api.py:306, 426, 457`: expands a single multi-antecedent SL justification into one SL per antecedent, i.e. an explicit OR. This is a convenience that makes the OR-of-ANDs structure visible to callers without hand-writing N justifications.

So §§1–3 are, in the code's reading, the definitional core of a TMS: what a *justification* is, what *SL* and *CP* mean, and what a *valid* justification entails. The paper's contribution that the code leans on hard is that retraction/restoration are **graph operations on this justification structure**, not ad-hoc bookkeeping — see `README.md:13` ("when a node goes OUT, all dependents whose justifications become invalid go OUT too") and `CLAUDE.md:13–14` ("retracted nodes stay in the DB — Doyle's insight: enables restoration without rederivation").

---

## §6 — dialectical framing, as cited in `tests/test_dialectical.py`

`tests/test_dialectical.py` starts with a docstring that is the only explicit "§6" claim in the observations:

```
Doyle Section 6: arguments become explicit, challengeable beliefs.
Instead of directly justifying N with evidence, you can challenge the
justification and create a dialectical chain.
```

What this means operationally, mapped to the tests:

### Challenge (`TestChallenge`, lines 11–90)
- `net.challenge(target, reason)` (test line 15) makes the target go OUT. This is asserted for both premises (`test_challenge_premise`, line 15) and derived nodes (`test_challenge_derived`, line 26).
- The challenge is a **first-class node**: `test_challenge_creates_node` (line 47) asserts a node `challenge-a` exists with `.text == "A is wrong because X"` and `.truth_value == "IN"`. This is exactly the docstring's claim — the *argument* becomes a *belief*.
- Bidirectional metadata is written: `challenge-a.metadata["challenge_target"] == "a"` and `"challenge-a" in a.metadata["challenges"]` (`test_challenge_metadata`, line 57).
- ID scheme: default `challenge-<target>`; repeated challenges get `-2`, `-3` suffixes (`test_multiple_challenges`, line 71).
- The challenge is a *defeater*: because `challenge-a` is IN, `a` is OUT. The natural reading (consistent with `README.md:15`'s "outlist … enables dialectical argumentation") is that the challenge is wired into a's justification as an outlist entry, so SL's "all outlist must be OUT" rule forces `a` OUT while `challenge-a` is IN.

### Defend (`TestDefend`, lines 96–160)
- `net.defend(target, challenge_id, reason)` (test line 100) restores the target to IN and places the challenge OUT. The defense itself is a node `defense-challenge-a` with metadata `defense_target == "challenge-a"` and `defends == "a"` (line 104).
- Defense is **not a separate primitive** in the Truth Maintenance sense — it is the same trick applied one level up: the defense node is the second-order defeater of the challenge node, which restores the target by restoring the challenge to OUT.

### Chains (`TestDialecticalChain`, lines 165–269)
- `test_challenge_defense` (line 168) walks: challenge A → A OUT; defend A → challenge OUT, A IN; challenge the defense → defense OUT, challenge restored, A OUT again. This is the "dialectical chain" — *challenge a defense*, then in `test_three_level_chain` (line 192) *defend the defense*, ad infinitum.
- `retract` is symmetric with all of the above: `test_dismiss_challenge_restores_target` (line 82) and `test_retract_defense_reinstates_challenge` (line 138) confirm that retraction of any node in the dialectical stack restores the node that was being defeated by it.

So the "§6 framing" is: **a justification is not just an assertion of support, it is a proposition that can itself be argued for or against.** The code implements this by treating defeats as just another SL link in the same graph, which is why the whole mechanism falls out of the retraction/restoration engine described in §§1–3.

---

## What the observations **do not** let us verify

- **Actual section numbering in Doyle (1979).** The claim that SL/CP are §1–3 and the dialectical framing is §6 comes *from the prompt and the test docstring*, not from a copy of the paper. The README cites only the whole paper.
- **Whether Doyle §6 literally describes challenge/defend**, or whether the codebase is extrapolating from Doyle's "explicit arguments" concept. The test docstring asserts this; nothing in the code proves it.
- **The concrete wiring of challenge/defend in `reasons/network.py`.** All the tests exercise it, but the implementation file is not in the observations. In particular we don't see whether `challenge()` adds a new SL justification to the target, edits an outlist, or uses a separate "defeated" flag — the tests are behavioral only.
- **The full `test_dialectical.py`** — only the first 200 of 269 lines are shown, so the tail of `test_three_level_chain` and any classes past `TestDialecticalChain` are unread.

If you want to answer "does the paper really say this?", you have to fetch Doyle 1979 itself; the codebase can only tell you what it *believes* the paper says.

---

## Topics to Explore

- [file] `reasons/network.py` — Contains the `challenge`/`defend` implementation; the tests pin its contract, but you need to read it to see whether challenges are encoded as outlist entries, extra justifications, or a separate defeat graph.
- [file] `reasons/__init__.py` — The `Node` and `Justification` dataclasses: read this to confirm the exact `SL`/`CP` semantics and the inlist/outlist shape the rest of the code branches on.
- [function] `reasons/api.py:add` — Convenience wrapper around `Justification` construction (`api.py:326–334`); shows how `--sl`, `--cp`, `--unless`, and `any_mode` are lowered into the primitive model.
- [general] `doyle-1979-paper` — Download or find the Doyle (1979) PDF to check that §6 actually contains the challenge/defend framing the test docstring attributes to it.
- [file] `tests/test_backtracking.py` — `README.md:194` and `CLAUDE.md:41` say entrenchment/trace/culprit-finding live here; it's the natural complement to the dialectical tests and probably shares the same justification graph.

## Beliefs

- `sl-requires-all-antecedents-in` — An SL justification is valid iff every antecedent is IN and every outlist node is OUT (`README.md:9,15`, `reasons/__init__.py:11`).
- `cp-justification-type-is-cp` — `Justification.type` is a plain string that takes the value `"CP"` (or `"SL"`) and is set by `--cp`/`--sl` in `api.py:331,334,444`.
- `challenge-creates-challenge-node` — `Network.challenge(target, reason)` creates a node with id `challenge-<target>` (suffixed `-2`, `-3` on repetition) that is IN and carries `metadata["challenge_target"] == target` (`tests/test_dialectical.py:47,57,71`).
- `challenge-requires-outlist-defeat` — Because a challenge places the target OUT while `challenge-<target>` is IN, the dialectical mechanism must be lowering challenges into the SL/outlist defeat layer described in `README.md:15`.
- `retract-challenge-restores-target` — `net.retract("challenge-<target>")` restores the challenged target to IN, and `net.retract("defense-<challenge>")` reinstates the challenge (making the target OUT again) (`tests/test_dialectical.py:82,138`).
- `dialectical-tests-cite-doyle-section-6` — The §6 attribution for challenge/defend appears only in the `tests/test_dialectical.py` docstring, not in any production module or the README.
