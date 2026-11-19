---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How ANY justifications are represented in the underlying storage versus the ALL-only assumption in this prompt (`REVIEW_JUSTIFICATIONS_PROMPT` says "currently all use ALL"); this assumption may become wrong as ANY storage is introduced.

## How ANY justifications are represented in storage versus the ALL-only assumption

The key distinction is that **ANY is not currently stored as an explicit justification mode**. Instead, it is represented structurally as multiple independent SL justifications.

### ALL storage shape

An ALL/conjunctive justification is stored as **one SL justification with multiple antecedents**.

Evidence from `tests/test_any_mode.py`:

- `TestAnyModeAdd::test_without_any_single_justification`:
  - `api.add_node("x", "Conclusion", sl="a,b,c", db_path=db)`
  - asserts `len(node["justifications"]) == 1`
  - asserts `len(node["justifications"][0]["antecedents"]) == 3`

So ALL means: one SL row where all antecedents must be IN.

### ANY storage shape

ANY is implemented by **expanding one logical ANY relation into one SL justification per antecedent**. Each resulting SL justification has a single antecedent. The disjunction comes from the TMS rule that the node is IN if any justification is satisfied.

Evidence from `tests/test_any_mode.py`:

- `TestAnyModeAdd::test_any_creates_separate_justifications`:
  - `api.add_node("x", "Conclusion", sl="a,b,c", any_mode=True, db_path=db)`
  - asserts `len(node["justifications"]) == 3`
  - asserts every justification has `len(j["antecedents"]) == 1`

- `TestAnyModeAdd::test_any_preserves_outlist`:
  - adds `sl="a,b", unless="enemy", any_mode=True`
  - asserts two justifications are created and each keeps `"enemy"` in its `outlist`

- `TestAnyModeAddJustification::test_any_expands_add_justification`:
  - `api.add_justification("x", sl="b,c", any_mode=True, db_path=db)`
  - appends separate SL justifications rather than creating a special ANY justification

- `TestPremiseCountWarning::test_add_any_returns_low_premise_count`:
  - with `any_mode=True`, returned `premise_count` is `1` because each generated justification has one premise

So the storage model appears to be:

| Logical mode | Stored representation |
|---|---|
| ALL | One SL justification with all antecedents |
| ANY | Multiple SL justifications, each with one antecedent |
| MIXED | Not a stored mode; could be approximated by a mix of multi-antecedent and single-antecedent SL justifications |

The observations do not show an explicit `mode`, `ANY`, or `ALL` field in the schema. Both schema searches:

- `inspect_storage_justification_representation`
- `inspect_pg_justification_schema_and_queries`

returned zero matches for `justification|ANY|ALL|mode`, so the exact database schema is not visible here. However, the API-level tests clearly establish the current representation.

## Why the prompt assumption is ALL-only

`reasons/review_justifications.py:16` defines `REVIEW_JUSTIFICATIONS_PROMPT`, which states:

> “Currently, all use ALL (conjunctive) justifications — meaning every antecedent must be IN for the conclusion to hold.”

That prompt is formatted and sent in `_process_batch` at `reasons/review_justifications.py:103`:

```python
prompt = REVIEW_JUSTIFICATIONS_PROMPT.format(beliefs=beliefs_text)
```

The review command filters candidates with `_has_multi_antecedent_sl(node, min_antecedents)`. By default, `min_antecedents=2`. That means `review_justifications()` only reviews nodes that have an SL justification with at least two antecedents.

This creates an important relationship:

- Nodes created with `any_mode=True` have only single-antecedent SL justifications.
- Therefore, `_has_multi_antecedent_sl` excludes them by default.
- Therefore, the prompt usually reviews only multi-antecedent SL rows, which today correspond to ALL-style storage.

So the prompt’s “currently all use ALL” is a **selection heuristic**, not a storage invariant. It is not enforced by a mode field in storage.

## Where this assumption can become wrong

The current assumption can break if ANY storage changes in any of these ways:

1. **An explicit mode field is added**
   - Example: one SL row with `antecedents=["a", "b", "c"]` plus `mode="ANY"`.
   - `_has_multi_antecedent_sl` would still select it, and the prompt would falsely tell the LLM that it is ALL.

2. **A multi-antecedent SL row is used to represent an ANY relation**
   - Today ANY expands to single-antecedent rows.
   - If future code stores ANY as a multi-antecedent row, the review filter would include it and the prompt would misclassify its starting assumption.

3. **Mixed justifications become common**
   - A node can already have multiple SL justifications, e.g. an original `SL(a)` plus `add_justification(..., any_mode=True)`.
   - The prompt text says each belief “was derived from its antecedents” and “all use ALL”, which may not describe a node with multiple justifications accurately.
   - `format_belief_for_review` in `reasons/review.py` controls what the LLM actually sees, but that function is not included in the observations.

4. **The parser defaults to ALL**
   - `parse_justification_review` in `reasons/review_justifications.py` defaults missing or non-string classification values to `"ALL"`:
     ```python
     "classification": cls.upper() if isinstance(cls, str) else "ALL",
     ```
   - This is another place where ALL is treated as the safe default. If ANY becomes a first-class stored mode, parser and prompt assumptions should be revisited.

## What is missing

The exact storage schema is not directly visible from the observations. The searches for schema terms returned no matches. To fully confirm the database-level representation, we would need:

- the actual table definitions or migrations for justifications
- the implementation of `api.add_node` and `api.add_justification` around `any_mode=True`
- the implementation of `format_belief_for_review` in `reasons/review.py`

The tests are sufficient to describe the current API-level behavior, but not to prove the raw database schema.

## Topics to Explore

- [file] `reasons/api.py` — Inspect `add_node` and `add_justification` around `any_mode=True` to see exactly where ANY is expanded into multiple SL rows.
- [function] `reasons/api.py:add_justification` — Defined at `reasons/api.py:406`; check whether any mode metadata is ever written into storage.
- [function] `reasons/review_justifications.py:_has_multi_antecedent_sl` — This filter determines which nodes the ALL-only prompt ever sees, and currently excludes ANY-expanded single-antecedent justifications.
- [file] `reasons/review.py` — `format_belief_for_review` determines what justification information is shown to the LLM, which matters for nodes with multiple justifications.
- [general] `justification-storage-schema` — Locate the actual DB schema/migrations to confirm whether there is any explicit `mode`, `ANY`, or `ALL` column.

## Beliefs

- `any-mode-produces-multiple-single-antecedent-sl-justifications` — With `any_mode=True`, the API creates one SL justification per premise, each containing one antecedent.
- `all-mode-stored-as-one-multi-antecedent-sl-justification` — Without `any_mode`, multiple premises are stored in a single SL justification with all antecedents in its `antecedents` list.
- `review-justifications-filters-out-any-expanded-nodes-by-default` — `_has_multi_antecedent_sl` with default `min_antecedents=2` excludes nodes whose ANY justifications have one antecedent each.
- `review-justifications-prompt-assumes-reviewed-justifications-use-all` — `REVIEW_JUSTIFICATIONS_PROMPT` at `reasons/review_justifications.py:16` explicitly states that all reviewed justifications currently use ALL.
- `storage-representation-exposes-no-explicit-any-mode-field` — Based on the observed tests and schema searches, ANY is expressed by multiple SL justifications rather than a stored `mode="ANY"` field.
