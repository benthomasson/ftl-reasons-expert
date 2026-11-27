---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Background on SL justifications and ALL/ANY semantics in the TMS model.

## Background: SL Justifications and ALL/ANY Semantics

### The Justification data model

Every node in the TMS carries zero or more `Justification` objects, defined at `reasons/__init__.py:7`. Each one has a `type` field that is either `"SL"` or `"CP"` (`reasons/__init__.py:19`), plus an `antecedents` list (the "inlist") and an `outlist`.

- **SL** (Support List) is the primary justification type (`reasons/__init__.py:11`):
  a node is `IN` **iff ALL antecedents are `IN` AND ALL outlist nodes are `OUT`** (`reasons/__init__.py:11-12`).
- **CP** is the other type mentioned in the same field but is not detailed by these observations (see Topics to Explore).

So a single SL is fundamentally **conjunctive**: every antecedent is required. Adding an antecedent makes the justification *harder* to satisfy. The outlist is the non-monotonic twist: the justification is only invalidated if an outlist node flips to `IN` (`reasons/__init__.py:12`, `.claude/skills/reasons/SKILL.md:37`).

### The node-level OR

Layered on top of that is a second, orthogonal OR: **a node is `IN` if ANY of its justifications is valid** (`reasons/__init__.py:30`). So the full truth condition for a derived node is:

```
truth(node) = OR over justifications j of (
    AND over antecedents a in j: truth(a) == IN
    AND AND over outlist o in j: truth(o) == OUT
)
```

This is the classic TMS structure: **conjunctive inside a justification, disjunctive across justifications**. Multiple justifications act as independent "routes" — retracting one antecedent only takes the node `OUT` if *every* route is broken (confirmed by `tests/test_add_justification.py:28-40`: node stays `IN` after retracting `a` because a second SL on `c` still holds).

### ALL vs ANY: what `any_mode` / `--any` does

The subtlety is that the *user-facing* CLI verb `add --sl a,b,c` does **not** mean "any of a, b, c suffices". By default it builds **one conjunctive SL with three antecedents**, requiring all three premises to be `IN` (`tests/test_any_mode.py:95-104`, `test_without_any_single_justification`).

Because that "ALL" default is easy to misread, the CLI offers an explicit `--any` flag (`reasons/cli.py:3076, 3079`) that performs a **structural decomposition**:

> `--any` expands an N-antecedent SL into N separate single-antecedent SL justifications.

Implementation points:
- `add_node(..., any_mode=...)` — `reasons/api.py:287, 306`. The expansion happens at `reasons/api.py:324-328`: when `any_mode and len(antecedents) > 1`, it loops over antecedents and appends one `Justification(type="SL", antecedents=[a], outlist=outlist, label=label)` per antecedent. Otherwise it appends the single conjunctive SL.
- `add_justification(..., any_mode=...)` — `reasons/api.py:413, 426`. Same expansion at `reasons/api.py:457-460`.
- Because the resulting justifications each carry a copy of the outlist, the non-monotonic guard is preserved per route (verified in `tests/test_any_mode.py:60-71`, `test_any_preserves_outlist`).

Semantically, `--any` converts:

```
SL(a) AND SL(b) AND SL(c)          # default: one string of ANDs
```

into

```
SL(a) OR SL(b) OR SL(c)            # --any: three independent routes
```

The observable consequences match this:
- Retracting one premise leaves the node `IN` (`tests/test_any_mode.py:32-42`, `test_any_survives_single_retraction`).
- Retracting all premises takes it `OUT` (`tests/test_any_mode.py:44-52`, `test_any_goes_out_when_all_retracted`).
- The returned `premise_count` drops to `1` under `--any`, because each justification now has a single premise (`tests/test_any_mode.py:145-153`, `test_add_any_returns_low_premise_count`).

### Guardrails: warnings, hints, and review

The codebase actively tries to catch mistaken ALL/ANY classification from three directions:

1. **Creation-time warning** — `reasons/cli.py:35-38` (`_warn_multi_premise`): if `premise_count >= 3` and `--any` was *not* passed, the CLI prints a Tip that "This SL requires ALL N premises to be IN. If any single…". This is the direct user protection against accidentally creating a 3-way conjunction.
2. **Retraction-time restoration hints** — `reasons/cli.py:129-132` (`_print_hints` style): when a multi-premise SL goes `OUT` but some premises survive, the CLI notes that the node went `OUT` because it required ALL of them. The mechanism and its `surviving_premises` / `all_premises` payload are pinned by `tests/test_any_mode.py:170-221` (`TestRestorationHints`). Crucially, nodes built with `--any` produce **no** hint because they never truly go `OUT` on a single retraction (`test_hint_with_any_flag_no_false_alarm`).
3. **Retrospective LLM review** — `reasons/api.py:4186-4218` exposes a review that asks a model whether each multi-antecedent SL "should be conjunctive (ALL) or disjunctive (ANY)" and tallies results into `convert_any` / `keep_all` (`reasons/api.py:4216-4218`). This is the only place where the ALL/ANY distinction is treated as a fallible inference rather than an API contract.

`reasons/derive.py:62` reinforces the same vocabulary at the derivation layer: it explicitly classifies each derivation as **ALL** or **ANY**, mirroring the two justification styles.

### Backend caveat

`any_mode` is **not supported on PostgreSQL**: both `add_node` (`reasons/api.py:313-314`) and `add_justification` (`reasons/api.py:432-433`) raise `NotImplementedError("any_mode is not supported with PostgreSQL")`. So on that backend, `--any` must be emulated by issuing multiple calls with a single antecedent each, or the user must accept a conjunctive SL.

### Note on missing information

The observations do not reveal: (a) the actual validity check implementation (the `AND` over antecedents and `OUT` over outlist evaluator lives somewhere in `reasons/network.py`, which is referenced by tests but never read here), (b) what CP justification means precisely beyond being an alternative `type` value, or (c) how the LLM reviewer at `reasons/api.py:4186+` is invoked (function name not captured). If the reader wants to reason about propagation internals, those need to be opened directly.

## Topics to Explore

- [function] `reasons/api.py:add_node` — Walks through the `any_mode` branch (lines 313–328) that turns a single `--sl` into N single-antecedent justifications
- [function] `reasons/api.py:add_justification` — Parallel expansion path at lines 432–460, plus the `len(j.antecedents) >= 2` handling at line 499
- [file] `reasons/network.py` — Contains the truth-value propagation that actually realizes the ALL-of-antecedents / ANY-of-justifications semantics
- [general] `cp-justification-semantics` — Understanding what `type="CP"` means and how it differs from SL (referenced at `reasons/__init__.py:19` and `tests/test_add_justification.py:141-148`)
- [general] `llm-all-any-review` — The review tool at `reasons/api.py:4186+` that audits multi-premise SLs for ALL/ANY misclassification

## Beliefs

- `sl-validity-requires-all-antecedents-in` — An SL justification is valid only if every antecedent is `IN` and every outlist node is `OUT` (`reasons/__init__.py:11-12`).
- `node-in-is-any-justification-valid` — A node's truth value is the OR of its justifications, so it stays `IN` while any one SL is valid (`reasons/__init__.py:30`).
- `any-mode-converts-all-to-any` — `any_mode`/`--any` expands a single N-antecedent SL into N separate single-antecedent SL justifications (`reasons/api.py:324-328`, `reasons/api.py:457-460`).
- `any-mode-is-unsupported-on-postgresql` — Both `add_node` and `add_justification` raise `NotImplementedError` when `any_mode=True` and the PostgreSQL backend is in use (`reasons/api.py:313-314`, `reasons/api.py:432-433`).
- `premise-defaults-to-empty-antecedent-sl` — A node added without an explicit `sl` still receives a trivially valid `type="SL"` justification with an empty antecedent list (`reasons/api.py:334`, `reasons/api.py:357-359`).
