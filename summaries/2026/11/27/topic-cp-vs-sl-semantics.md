---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `type="CP"` justifications behave differently from `type="SL"` at propagation, backtracking, or explanation time (observations suggest only a tag difference).

## Verdict

The available evidence strongly supports the claim that `type="CP"` is mostly a tag difference from `type="SL"` **at propagation, backtracking, and explanation time**, but I would not call it proven because the actual implementations of the propagation/backtracking functions were not included in the observations.

The clearest evidence is:

- Both are represented by the same `Justification` dataclass with a string `type` field: `"SL"` or `"CP"` (`reasons/__init__.py:7`, `reasons/__init__.py:19`).
- Both use the same `antecedents`, `outlist`, and `label` fields; no separate subclass or alternate data shape is shown.
- `reasons/network.py:982` handles both together with `if j.type in ("SL", "CP"):`, which suggests at least that branch does not distinguish them.
- Validation/explanation messages interpolate the type as a label only, e.g. `f"{j.type} justification valid"` / `f"{j.type} justification invalid"` (`reasons/network.py:867`, `reasons/network.py:890`; `reasons/pg.py:3479`, `reasons/pg.py:3514`).

So at explanation time, the type appears to affect only text/tagging, not logic.

## Where the types are constructed

`reasons/api.py:300-331` accepts `sl` vs `cp` inputs and creates `Justification(type="SL", ...)` or `Justification(type="CP", ...)` respectively. If no explicit justification is supplied, the default is SL (`reasons/api.py:334`, `reasons/api.py:357-359`).

There is one confirmed creation/API-time difference: `any_mode` expansion is explicitly SL-only:

- `reasons/api.py:306`: “If True, expand SL into one justification per antecedent”
- `reasons/api.py:457`: `if any_mode and jtype == "SL" and len(antecedents) > 1:`

So “tag only” is not literally true everywhere. `any_mode` specializes SL construction.

There is also an SL-specific multi-antecedent check at `reasons/api.py:499`:

```python
if j.type == "SL" and len(j.antecedents) >= 2:
```

The surrounding function is not included, so I cannot tell whether this is validation, review, migration, or diagnostic logic. It is outside the shown propagation/backtracking paths, but it is a real place where SL gets a branch CP does not.

## Propagation

The direct propagation code was not returned. The search pattern for `(propagat|backtrack|explain|explanation)` returned no matches, which is a gap in the observations.

However, the observed type checks suggest shared propagation behavior:

- `reasons/network.py:982` checks both SL and CP in the same condition.
- No observed propagation branch checks `j.type == "CP"` separately.
- The `Justification` model itself gives CP no extra fields beyond SL.

So the best-supported reading is: propagation walks the same antecedent graph for both, using `type` as metadata. But the actual `trace_assumptions`, `find_culprits`, `add_nogood`, and truth-propagation implementations are not visible, so this remains an inference rather than a fully verified fact.

## Backtracking

The backtracking tests in `tests/test_backtracking.py` exercise `trace_assumptions`, `find_culprits`, and `add_nogood`. All justifications used in those tests are `Justification(type="SL", ...)`. There are no CP-specific backtracking tests in the provided excerpt.

Those tests show that backtracking is driven by the antecedent graph:

- `trace_assumptions` walks from a derived node through `antecedents` back to premises.
- `find_culprits` identifies premises responsible for a contradiction.
- `add_nogood` retracts a premise using entrenchment/dependents logic.

Nothing in the tests suggests a CP/SL branch. But because the implementations are not shown and there are no CP backtracking cases, the observation set is insufficient to prove that CP backtracking is identical. It is likely identical if, as the model suggests, the only stored distinction is the tag.

## Explanation time

Observed explanation/validation messages do not appear to branch on CP vs SL semantically. They print the type:

- `reasons/network.py:867`: `"reason": f"{j.type} justification valid"`
- `reasons/network.py:890`: `"reason": f"{j.type} justification invalid"`
- `reasons/pg.py:3479`: `"reason": f"{jtype} justification valid"`
- `reasons/pg.py:3514`: `"reason": f"{jtype} justification invalid"`

That is consistent with the observation that CP vs SL is mostly a display/classification tag in those paths.

## What is missing

To verify the claim completely, we need the bodies of:

- `Network.trace_assumptions`
- `Network.find_culprits`
- `Network.add_nogood`
- the propagation/update logic that evaluates whether a node is IN/OUT
- any CP-specific “assumptions are consistent” implementation implied by `reasons/__init__.py:13`

Also, `reasons/api.py:499` needs surrounding context to understand whether the SL-only multi-antecedent branch affects runtime semantics or only diagnostic/review behavior.

## Topics to Explore

- [file] `reasons/network.py` — Contains the actual `Network` propagation and backtracking implementation; inspect `trace_assumptions`, `find_culprits`, and `add_nogood` for explicit `j.type` branches.
- [function] `reasons/network.py:trace_assumptions` — Confirm whether it follows `antecedents` generically or treats CP justifications differently.
- [function] `reasons/network.py:add_nogood` — Verify whether victim selection/backtracking looks at justification type or only at graph structure and entrenchment.
- [file] `reasons/api.py` — Inspect the context around lines 300-500, especially the SL-only `any_mode` and multi-antecedent checks.
- [general] `CP-consistency-check` — Search for the implementation of the CP semantic “node is IN iff assumptions are consistent” described at `reasons/__init__.py:13`.

## Beliefs

- `justification-type-is-string-tag` — `Justification.type` is a string field holding `"SL"` or `"CP"`; there is no separate CP/SL subclass in the observed definition.
- `CP-and-SL-share-antecedent-model` — Both CP and SL justifications use the same `antecedents`, `outlist`, and `label` fields.
- `any-mode-is-sl-only` — `any_mode` expansion is guarded by `jtype == "SL"` and does not apply to CP (`reasons/api.py:457`).
- `network-validity-handles-cp-and-sl-together` — `reasons/network.py:982` checks `j.type in ("SL", "CP")`, indicating shared handling in that branch.
- `explanations-tag-type-without-observed-branching` — Validation/explanation messages interpolate `j.type`/`jtype` but do not show CP-specific logic (`reasons/network.py:867`, `reasons/network.py:890`; `reasons/pg.py:3479`, `reasons/pg.py:3514`).
