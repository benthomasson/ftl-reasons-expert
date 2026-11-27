---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `CP` (conditional proof) should be treated as logically distinct from `SL`; the current code treats them identically, which may be either simplification or a bug.

# Should `CP` be logically distinct from `SL`?

## Verdict

**Yes, in principle `CP` should be distinct — but in this codebase it currently isn't.** The truth calculus treats `SL` and `CP` identically (`reasons/network.py:982`), while the documentation promises a different truth condition for `CP` ("assumptions are consistent" vs "all antecedents are IN"). Combined with the fact that a few peripheral code paths *do* gate on `type == "SL"` specifically, the most likely reading is that `CP` is an **unfinished feature / latent bug**, not an intentional simplification.

## Where the code treats them identically (the crux)

- `reasons/network.py:982` — `if j.type in ("SL", "CP"):` — both justification types flow through the same truth-evaluation branch. This is the decision point that determines whether a node is IN/OUT, and it does not distinguish the two.
- `reasons/pg.py:45` — `CHECK (type IN ('SL', 'CP'))` — the schema enforces the two-type vocabulary but does not encode any behavioral difference.
- `reasons/api.py:331` and `:444` — both construct `Justification(type=..., antecedents=..., outlist=..., label=...)` with the same data shape. Nothing in the observed code performs a "consistency" check before storing a `CP` justification.

## What the docs promise

- `reasons/__init__.py:11–13`:
  - SL: "node is IN iff ALL antecedents (inlist) are IN"
  - CP: "node is IN iff assumptions are consistent"
- `README.md:167` — `--cp` usage: "CP justification (assumptions must be consistent)".
- `CLAUDE.md:11` — only documents `SL`; `CP` is absent from the primary justification description.

The word **"consistent" is never defined** in any observed code or doc, and no observed code checks it. In classical logic, Conditional Proof means: assume `P`, derive `Q`, discharge the assumption to conclude `P → Q`. That requires scoped assumptions and a discharge mechanism. The `Justification` model here (`antecedents` + `outlist`) has **no scope field, no subproof tracking, and no discharge operation** — so the classical CP machinery is absent even if the naming evokes it.

## Where the code *does* distinguish them (evidence against pure simplification)

Two peripheral paths treat `CP` as a different animal, which argues the author intended a distinction:

- `reasons/api.py:457` — `if any_mode and jtype == "SL" and len(antecedents) > 1:` — the `--any` OR-mode expansion applies only to `SL`. This makes sense: you wouldn't want to split a `CP` justification into per-antecedent disjuncts.
- `reasons/api.py:499` — `if j.type == "SL" and len(j.antecedents) >= 2:` — the "review SL justifications" analysis explicitly skips `CP` nodes.

These are feature-coverage asymmetries rather than truth-condition differences, but they show the types are not simply synonyms in every code path.

## Is it a bug or a simplification?

Evidence for **bug / unfinished feature**:

1. `pg.py:45` and `__init__.py:19` enforce a two-value vocabulary — the author made the type first-class.
2. `__init__.py:13` states a distinct truth condition that no visible code implements.
3. `api.py:457` and `api.py:499` explicitly gate SL-only behavior, confirming the author's mental model was "these are different kinds of justifications."
4. Test coverage — `tests/test_add_justification.py:167` only asserts `type == "CP"` was stored. There is **no test asserting a CP node flips IN/OUT under different circumstances than an SL node**.
5. `CLAUDE.md` and the `reasons` skill (`.claude/skills/reasons/SKILL.md:33`) document only SL — CP is a documented-but-orphaned type.

The consequence for a user: `reasons add X "text" --cp a,b` and `reasons add X "text" --sl a,b` produce **identical truth behavior**, despite the README promising different semantics. If `__init__.py:13` is the spec, this is a bug. If "CP is an annotation flavor of SL" is the intent, the docstring is stale — but no visible artifact reconciles those possibilities.

## What's missing to be definitive

- The **full body** of `reasons/api.py:add_justification` (around lines 300–345 and 420–470) to confirm no consistency check runs before the CP is stored. The observations only show the parameter docstring and the append.
- The **full body around `reasons/network.py:982`** to see if any adjacent branch or comment either justifies the lumping or contradicts it.
- Any **test** that exercises CP truth semantics (not just storage), beyond `test_add_justification.py:167`.
- A **design note or changelog entry** defining what "assumptions are consistent" was meant to mean operationally in this TMS.

## Topics to Explore

- [function] `reasons/network.py` — Locate the function containing line 982 (`if j.type in ("SL", "CP")`) to see the full truth-evaluation branch, and confirm whether any CP-specific comment or sibling check exists nearby.
- [function] `reasons/api.py:add_justification` — Read the full body to verify whether a "consistency" check is performed before a CP justification is stored, or whether CP is passed straight through to the Justification model.
- [file] `reasons/__init__.py` — Inspect the `Justification` dataclass definition (`__init__.py:7`) to confirm there is no scope/assumption field that could support classical CP discharge.
- [general] `cp-consistency-implementation` — Search the whole repo for any code that computes "consistency of assumptions" — the observations suggest none exists, but a repo-wide grep would settle it.
- [file] `tests/test_add_justification.py` — Determine whether any test exercises CP *behavior* (IN/OUT transitions) versus only CP *storage* (`type == "CP"` assertion at line 167).

## Beliefs

- `cp-and-sl-share-truth-evaluation` — `reasons/network.py:982` maps both `j.type == "SL"` and `j.type == "CP"` into the same truth-evaluation branch, so CP nodes currently inherit SL's "all antecedents IN" semantics.
- `cp-docstring-promises-consistency-semantics` — `reasons/__init__.py:13` and `README.md:167` document CP as "node is IN iff assumptions are consistent," a truth condition distinct from SL's "all antecedents IN."
- `cp-any-mode-not-supported` — `reasons/api.py:457` restricts the `--any` OR-mode expansion to `jtype == "SL"`, so `--cp a,b` cannot be split into per-antecedent disjuncts.
- `cp-excluded-from-sl-review-path` — `reasons/api.py:499` filters on `j.type == "SL" and len(j.antecedents) >= 2`, so multi-antecedent CP justifications are skipped by the SL-review analysis.
- `cp-type-is-schema-enforced` — `reasons/pg.py:45` enforces `type IN ('SL', 'CP')` at the database layer, making the two types a first-class vocabulary even though their truth behavior is identical.
- `cp-test-coverage-is-storage-only` — `tests/test_add_justification.py:167` asserts only `justifications[1]["type"] == "CP"`; no observed test asserts CP behaves differently from SL when antecedents change state.
