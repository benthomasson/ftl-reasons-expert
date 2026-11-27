---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `mode="any"` interacts with outlist semantics (currently not exercised by any observed test).

## Short answer

Yes. `mode="any"` **does interact with `outlist` semantics**, but the interaction is mechanical via **justification cloning** rather than a special OR+unless evaluator.

For SL justifications, `any_mode=True` expands one multi-antecedent SL into one SL justification per antecedent. The same `outlist` is attached to **every** generated justification:

- `reasons/api.py:add_node` parses `unless` into `outlist` at `api.py:320`, then for `any_mode` and `len(antecedents) > 1` creates one `Justification(type="SL", antecedents=[a], outlist=outlist, ...)` per antecedent at `api.py:324-326`.
- `reasons/api.py:add_justification` does the same at `api.py:437` and `api.py:457-460`.

Semantically, this makes the outlist a **shared veto across all OR branches**. Each expanded justification is valid only when its single antecedent is IN and all outlist nodes are OUT. The validity check seen in the observations computes:

```python
outlist_ok = all(
    ...
    for o in j.outlist
)
```

at `reasons/api.py:1590-1592`, implying outlist validity is evaluated per justification.

So:

```text
x if a,b any_mode unless enemy
```

becomes conceptually:

```text
x if a unless enemy
x if b unless enemy
```

That is equivalent to:

```text
(a OR b) AND enemy is OUT
```

not:

```text
(a AND enemy is OUT) OR b
```

Therefore, if `enemy` is IN, **all** expanded `any_mode` justifications for that call are defeated, even if `a` or `b` is IN. If `enemy` is OUT, each branch can independently keep `x` IN. Multiple outlist entries must all be OUT for each expanded justification to hold. Other independent justifications on the same node can still keep it IN, matching normal multiple-justification semantics.

## Test coverage

There **is** a structural test:

- `tests/test_any_mode.py::TestAnyModeAdd::test_any_preserves_outlist` (call around `tests/test_any_mode.py:63`) asserts that each expanded justification contains `"enemy"` in its `outlist`.

That test confirms the outlist is copied to every expanded justification. However, the observations do **not** show a test that asserts the semantic interaction, such as:

- `any_mode=True` + `unless` where the outlist node is IN causes the node to go OUT.
- Retracting the outlist node restores the node via one of the remaining IN antecedents.
- Multiple outlist entries with `any_mode=True` require all outlist nodes to be OUT.
- `add_justification(..., any_mode=True, unless=...)` goes OUT/IN correctly as the outlist node changes.
- A derive `GATE` proposal with `Mode: ANY` and `Unless:` applies with the expected outlist-veto behavior.

The existing any-mode tests (`tests/test_any_mode.py`) mostly exercise OR expansion and retraction of antecedents. The existing outlist tests (`tests/test_outlist.py`) exercise outlist semantics but not in combination with `any_mode`.

So the statement “currently not exercised by any observed test” is **mostly true for semantic interaction**, but should be qualified: the outlist is structurally preserved by `any_mode`, and that specific preservation is tested.

## Derive path caveat

`reasons/derive.py` parses proposals with both `Unless` and `Mode: ALL/ANY` in `parse_proposals`, and `reasons/derive.py:664` maps `mode == "any"` into `any_mode=p.get("mode") == "any"`. That suggests derive-applied `GATE` + `Mode: ANY` + `Unless` should feed into the same API behavior.

However, the observations only include the first 600 lines of `reasons/derive.py`, while line 664 is outside that excerpt. So the exact `apply_proposals` wiring is not fully visible. The API-level behavior is clear; the derive-specific end-to-end behavior is not fully verified from the provided observations.

Also note: PostgreSQL explicitly rejects `any_mode` at `api.py:313-314` and `api.py:432-433`, so this interaction is only relevant to the SQLite/network-backed path.

## Topics to Explore

- [file] `reasons/api.py` — Inspect `add_node` and `add_justification` around the `any_mode` and `outlist` branches, especially `api.py:320-326` and `api.py:437-460`, to confirm cloning and edge cases.
- [function] `reasons/api.py:add_node` — Verify that `any_mode` only expands SL, while CP and outlist-only justifications ignore it.
- [function] `reasons/api.py:add_justification` — Verify the same expansion rules and note that the returned result comes from the final added justification in the `any_mode` loop.
- [file] `tests/test_any_mode.py` — Add the missing semantic tests for `any_mode=True` plus `unless`, including outlist-IN defeat, outlist-OUT restoration, and multiple outlist entries.
- [file] `reasons/derive.py` — Inspect `apply_proposals` around `derive.py:664` and the GATE proposal path to confirm `Mode: ANY` plus `Unless:` is passed correctly into the API.

## Beliefs

- `any-mode-clones-outlist-per-expanded-justification` — `add_node` and `add_justification` attach the same outlist to every SL justification generated by `any_mode`.
- `any-mode-outlist-acts-as-shared-veto` — Because each expanded justification carries the same outlist, an IN outlist node defeats all OR branches generated by that call.
- `any-mode-does-not-apply-to-cp-justifications` — `any_mode` expansion is limited to SL; CP justifications are created with the full antecedent list regardless of `any_mode`.
- `postgresql-rejects-any-mode` — `any_mode=True` raises `NotImplementedError` when a PostgreSQL connection is supplied.
- `any-mode-outlist-semantics-is-untested` — Observed tests assert structural outlist preservation under `any_mode`, but not the truth-value semantics when outlist nodes change IN/OUT.
