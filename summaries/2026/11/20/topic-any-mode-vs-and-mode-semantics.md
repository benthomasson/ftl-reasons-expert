---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How `sl="a,b,c"` is interpreted as AND (one conjunctive JSL) vs. OR (three unary JSLs); this distinction drives the entire #20 feature and shapes the CLI UX.

## How `sl="a,b,c"` becomes AND vs. OR

The key is whether the comma-separated SL is stored as **one justification with many antecedents** or **many justifications with one antecedent each**.

### 1. Default behavior: `sl="a,b,c"` means AND

Without `any_mode`, `api.add_node(..., sl="a,b,c")` treats `a,b,c` as the antecedents of a single SL justification.

- `reasons/api.py` `add_node` accepts `any_mode: bool = False` at line 287.
- The expansion branch is guarded by:

  ```python
  if any_mode and len(antecedents) > 1:
  ```

  at `reasons/api.py:324`.

So with `any_mode=False`, that branch is skipped. The result is one justification:

```text
SL(a, b, c)
```

This is conjunctive: the node is supported only if **all** antecedents in that justification are IN.

The test `test_without_any_single_justification` in `tests/test_any_mode.py` confirms this:

- `len(node["justifications"]) == 1`
- `len(node["justifications"][0]["antecedents"]) == 3`

That is the default AND interpretation.

### 2. `any_mode=True`: `sl="a,b,c"` means OR

With `any_mode=True`, the same parsed antecedent list hits the expansion branch at `reasons/api.py:324`.

Instead of one justification:

```text
SL(a, b, c)
```

the API creates three unary justifications:

```text
SL(a)
SL(b)
SL(c)
```

Each one has exactly one antecedent. At the node level, these justifications are alternatives: the node can remain IN if any one of them is satisfied.

The test `test_any_creates_separate_justifications` confirms:

- `len(node["justifications"]) == 3`
- every justification has `len(j["antecedents"]) == 1`

And the retraction tests show the semantic difference:

- `test_any_survives_single_retraction`: retracting `a` leaves `x` IN because `SL(b)` and `SL(c)` still hold.
- `test_any_goes_out_when_all_retracted`: after retracting all supporting premises, `x` goes OUT.

So `--any` converts a single conjunctive SL into multiple disjunctive alternatives.

### 3. Single-premise SLs are not expanded

The expansion only happens when `len(antecedents) > 1`.

`test_any_with_single_premise_no_expansion` verifies that:

```python
api.add_node("x", "Conclusion", sl="a", any_mode=True)
```

still produces exactly one justification. There is no useful OR expansion for a single premise.

### 4. `add_justification` follows the same rule

`api.add_justification` also accepts `any_mode` at `reasons/api.py:413`.

Its expansion branch is at `reasons/api.py:457`:

```python
if any_mode and jtype == "SL" and len(antecedents) > 1:
```

So `add_justification("x", sl="b,c", any_mode=True)` creates separate unary SL justifications for `b` and `c`, instead of one conjunctive `SL(b,c)` justification.

This is covered by:

- `test_any_expands_add_justification`
- `test_any_add_justification_restores_out_node`

In the second test, retracting `a` makes `x` OUT, then adding `--any` justifications for `b,c` restores it to IN.

### 5. PostgreSQL does not support `any_mode`

Both `add_node` and `add_justification` explicitly reject `any_mode` for PostgreSQL:

- `reasons/api.py:313-314`
- `reasons/api.py:432-433`

They raise:

```python
NotImplementedError("any_mode is not supported with PostgreSQL")
```

So the `--any` behavior is currently a SQLite-side capability.

## How this drives feature #20 and the CLI UX

Feature #20 is exactly the `--any` flag: it lets the user choose OR semantics instead of the default AND semantics.

### CLI passes `--any` into the API

In `reasons/cli.py`:

- `cmd_add` passes `any_mode=getattr(args, "any", False)` at `reasons/cli.py:57`.
- `cmd_add_justification` passes it at `reasons/cli.py:80`.

The argparse flags are defined with help text:

```python
p.add_argument(
    "--any",
    action="store_true",
    help="Expand SL into one justification per premise (OR instead of AND)"
)
```

at `reasons/cli.py:3079` and `reasons/cli.py:3097`.

That help text is the clearest UX statement of the distinction.

### The 3+ premise warning is shaped by `premise_count`

`_warn_multi_premise` at `reasons/cli.py:35-39` prints a tip when:

```python
if premise_count >= 3 and not any_mode:
```

It warns:

```text
Tip: This SL requires ALL 3 premises to be IN. If any single
premise is sufficient, use --any to create separate justifications.
```

The API returns `premise_count` for this purpose:

- `test_add_returns_premise_count`: default `sl="a,b,c"` returns `premise_count == 3`.
- `test_add_any_returns_low_premise_count`: with `any_mode=True`, returns `premise_count == 1`, because each justification has one premise.

So the CLI warning naturally disappears under `--any`: the user has already chosen the OR interpretation.

### Restoration hints show how to recover from AND cascades

When a multi-premise AND justification cascades OUT, the CLI can print a restoration hint. In `reasons/cli.py`, `_print_restoration_hints` prints:

```text
reasons add-justification {hint['node_id']} --sl {surviving} --any
```

at `reasons/cli.py:136`.

The same pattern appears around `reasons/cli.py:2543`.

This directly ties the AND/OR distinction to the CLI workflow:

1. A node went OUT because its single justification required all premises.
2. Some premises are still IN.
3. If any surviving premise should be sufficient, the user can add a new justification with `--any`.
4. That creates unary justifications from the surviving premises.

### The derive workflow also uses the same concept

`reasons/derive.py` parses LLM proposals with a `Mode: ALL` or `Mode: ANY` field. At `reasons/derive.py:664`, it maps:

```python
any_mode=p.get("mode") == "any"
```

So the same semantic distinction is used not only for manual CLI adds, but also for LLM-proposed derivations.

## What the observations do not show

The exact parsing function that splits `sl="a,b,c"` into an antecedent list is not shown in the provided slices. The tests and API branches make the semantics clear, but the precise helper that tokenizes the string is not visible here.

Also, the full truth-value evaluation engine is not shown. The tests establish the behavior: multiple justifications behave disjunctively, while antecedents inside one justification behave conjunctively.

## Topics to Explore

- [file] `reasons/api.py` — Inspect `add_node` and `add_justification` around lines 287-324 and 413-457 to see the exact expansion implementation.
- [file] `tests/test_any_mode.py` — Full coverage of OR survival, single-premise no-expansion, outlist preservation, and add-justification restoration.
- [function] `reasons/cli.py:_warn_multi_premise` — Shows how `premise_count` and `any_mode` drive the user-facing 3+ premise tip.
- [file] `reasons/derive.py` — See how `Mode: ALL|ANY` from LLM proposals maps into `any_mode` at line 664.
- [general] `truth-value-evaluation` — Explore how the RMS evaluates multiple justifications and antecedents to decide IN/OUT.

## Beliefs

- `default-sl-with-three-antecedents-produces-one-conjunctive-justification` — Without `any_mode`, `sl="a,b,c"` creates one SL justification with three antecedents.
- `any-mode-expands-multi-premise-sl-into-unary-justifications` — With `any_mode=True`, `sl="a,b,c"` creates three justifications, each with one antecedent.
- `any-mode-expansion-requires-more-than-one-antecedent` — The expansion branch only runs when `len(antecedents) > 1`, so `sl="a"` remains a single justification.
- `any-mode-is-rejected-by-postgresql-backend` — Both `api.add_node` and `api.add_justification` raise `NotImplementedError` when `any_mode` is used with PostgreSQL.
- `cli-multi-premise-warning-is-suppressed-by-any` — `_warn_multi_premise` prints only when `premise_count >= 3` and `any_mode` is false.
