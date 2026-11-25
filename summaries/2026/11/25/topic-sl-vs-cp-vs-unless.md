---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Precise roles of the three support-list parameters in the belief semantics and how they combine under `any_mode`

## Belief semantics: the support-list parameters

In the observed code, a justification of type `SL` is evaluated against two kinds of node lists:

- **Positive support / antecedents**: nodes that must be `IN` for the justification to hold.
- **Negative support / outlist**: nodes that must be `OUT` for the justification to hold.

A node is justified when at least one of its justifications holds. For a single `SL` justification with antecedents `A` and outlist `U`, the condition is effectively:

```text
(AND_{a in A} IN(a)) AND (AND_{u in U} OUT(u))
```

The observed names are:

### 1. `sl` — positive support list

`sl` is the comma-separated support list that becomes the justification’s `antecedents`.

- In normal mode (`any_mode=False`), multiple entries in `sl` form one conjunctive justification: all antecedents must be `IN`.
- This is demonstrated by `tests/test_any_mode.py:71-78`: `sl="a,b,c"` without `any_mode` produces one justification with three antecedents.

So `sl` answers: **what must be true together to support this conclusion?**

### 2. `unless` — negative support list

`unless` is the API-facing way to specify nodes that defeat a justification. It is stored on the justification as `outlist`.

- If any node in the outlist is `IN`, that justification is defeated.
- `tests/test_any_mode.py:63-69` uses `unless="enemy"` and asserts that every created justification contains `"enemy"` in `j["outlist"]`.

So `unless` answers: **what must remain false/out for this justification to remain valid?**

### 3. `outlist` — stored negative-support field

In the observed tests, `outlist` is the justification field populated from `unless`, not an independent semantic concept.

- `api.add_node(..., unless="enemy", ...)` results in `j["outlist"] == ["enemy"]` in `tests/test_any_mode.py:63-69`.
- The exact public signatures around `reasons/api.py:287` and `reasons/api.py:413` are not fully shown, so the observations do not prove whether an additional `outlist=` API parameter exists. If it does exist, it should be treated as the full-form negative support list feeding the same justification field.

So in the observed behavior: **`unless` is the input, `outlist` is the stored justification data, and both represent negative support.**

---

## How `any_mode` combines with the support lists

`any_mode` changes only the positive `sl` side. It does not turn the negative support list into an OR condition.

### Normal mode

With:

```python
sl = "a,b,c"
unless = "enemy"
any_mode = False
```

you get one justification:

```text
a AND b AND c AND NOT enemy
```

This matches `tests/test_any_mode.py:71-78`, where a multi-premise `sl` without `any_mode` produces one justification with three antecedents.

### `any_mode=True`

With:

```python
sl = "a,b,c"
unless = "enemy"
any_mode = True
```

the `sl` list is expanded into one justification per antecedent:

```text
(a AND NOT enemy)
OR
(b AND NOT enemy)
OR
(c AND NOT enemy)
```

The outlist is copied into every expanded justification. That is exactly what `tests/test_any_mode.py:63-69` checks: two expanded justifications still each contain `"enemy"` in `j["outlist"]`.

The expansion is gated by:

- `reasons/api.py:324` for `add_node`: `if any_mode and len(antecedents) > 1:`
- `reasons/api.py:457` for `add_justification`: `if any_mode and jtype == "SL" and len(antecedents) > 1:`

So `any_mode` does not expand:

- single-premise `sl` values (`tests/test_any_mode.py:53-58`), or
- non-`SL` justification types in `add_justification`, because of the `jtype == "SL"` check.

### PostgreSQL limitation

`any_mode` is explicitly unsupported with PostgreSQL:

- `reasons/api.py:313-314`: `add_node` raises `NotImplementedError("any_mode is not supported with PostgreSQL")`
- `reasons/api.py:432-433`: `add_justification` raises the same error

### Consequence for `premise_count`

Because each expanded justification has exactly one antecedent, `any_mode=True` changes the reported premise count. `tests/test_any_mode.py:127-133` asserts that:

```python
api.add_node("x", "Conclusion", sl="a,b,c", any_mode=True)
```

returns `premise_count == 1`, not `3`.

### Consequence for restoration hints

Because `any_mode` justifications each have one premise, they do not trigger multi-premise restoration hints. `tests/test_any_mode.py:198-203` checks that retracting one premise from an `any_mode` node leaves the node `IN` via the other branches and produces no restoration hint.

---

## Important caveat about “three support-list parameters”

The observations clearly show:

- `sl`
- `unless`
- `outlist` as the stored justification field

They do **not** show the full signatures of `reasons/api.py:add_node` or `reasons/api.py:add_justification`, so the codebase may have an additional public parameter such as `outlist=` that is not visible in the snippets. If so, its likely role is to provide the full negative-support list directly, with `unless` being a convenience alias or single-node form. That relationship is not confirmed by the provided observations.

## Topics to Explore

- [function] `reasons/api.py:add_node` — Confirm the exact signature and how `sl`, `unless`, and any `outlist` parameter are parsed before justification creation.
- [function] `reasons/api.py:add_justification` — Inspect the `jtype == "SL"` gate and how non-SL justifications behave under `any_mode`.
- [file] `reasons/derive.py` — See how proposal `mode == "any"` maps into `any_mode` at line 664 and how derived justifications are applied.
- [file] `tests/test_any_mode.py` — Read the full behavioral contract for expansion, outlist preservation, `premise_count`, and restoration hints.
- [general] `belief-semantics` — Trace the truth-value evaluation for `SL` justifications to confirm exactly how antecedents and outlists determine `IN`/`OUT`.

## Beliefs

- `any-mode-expands-sl-into-or-justifications` — With `any_mode=True`, an `SL` justification with more than one antecedent is expanded into one justification per antecedent.
- `any-mode-preserves-outlist-per-expanded-justification` — Each expanded justification retains the original outlist, so `unless`/negative support applies to every OR branch.
- `any-mode-requires-sl-jtype-in-add-justification` — In `add_justification`, expansion is gated by `jtype == "SL"` and `len(antecedents) > 1` at `reasons/api.py:457`.
- `any-mode-reports-expanded-premise-count` — With `any_mode=True`, a multi-premise `sl` reports `premise_count == 1` because each generated justification has one antecedent.
- `any-mode-unsupported-with-postgresql` — Both `add_node` and `add_justification` raise `NotImplementedError` when `any_mode=True` with PostgreSQL (`reasons/api.py:313-314`, `reasons/api.py:432-433`).
