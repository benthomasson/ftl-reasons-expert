---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: How OR expansion is modeled (multiple single-antecedent justifications rather than a special `ANY` type), and why `premise_count` is hardcoded to `1` in that branch.

## How OR expansion is modeled

RMS does not model OR by introducing an `ANY` justification type. Instead, it relies on the existing truth-network semantics: a node is IN if **any** of its justifications is valid (`reasons/__init__.py:30`, `reasons/network.py:963`). A single SL justification remains conjunctive over its antecedents: all antecedents must be IN for that justification to be valid.

`any_mode` is an API/CLI convenience that rewrites one multi-premise SL into multiple single-premise SL justifications. The `add_node` docstring states this directly: `any_mode: If True, expand SL into one justification per antecedent (OR)` (`reasons/api.py:306`). The expansion is guarded by `if any_mode and len(antecedents) > 1:` (`reasons/api.py:324`). `add_justification` does the same for SL justifications with more than one antecedent via `if any_mode and jtype == "SL" and len(antecedents) > 1:` (`reasons/api.py:457`).

So:

- `sl="a,b,c"` without `any_mode` → one SL justification with antecedents `[a, b, c]` (ALL).
- `sl="a,b,c"` with `any_mode=True` → three SL justifications: `[a]`, `[b]`, `[c]` (OR).

The tests establish this contract. `tests/test_any_mode.py:22` creates a 3-premise any-mode node and asserts three justifications, each with one antecedent. `tests/test_any_mode.py:34` retracts one premise and expects the node to remain IN. `tests/test_any_mode.py:43` retracts all alternatives and expects OUT. `tests/test_any_mode.py:53` confirms a single premise is not expanded. `tests/test_any_mode.py:63` confirms the same outlist is copied to each expanded justification. `tests/test_any_mode.py:91` and `:107` cover the same behavior for `add_justification`.

This means there is no persisted `ANY` type, no schema change, and no special evaluator branch. The `ANY` strings that appear elsewhere are review/classification labels (`reasons/api.py:4186`, `reasons/api.py:4216`) or proposal input (`reasons/derive.py:62`, `reasons/derive.py:664`), not justification storage types. PostgreSQL explicitly rejects `any_mode`: both entry points raise `NotImplementedError` (`reasons/api.py:313`, `reasons/api.py:432`).

## Why `premise_count` is hardcoded to `1` in that branch

`premise_count` is not the total number of original premise nodes supplied by the caller. It is a warning/hint-oriented value describing the resulting justification shape. For ordinary `add_node`, the return uses `"premise_count": max_premises` (`reasons/api.py:400`). For ordinary `add_justification`, it uses `"premise_count": len(antecedents)` (`reasons/api.py:469`). In the any-mode `add_justification` branch, the code sets `result["premise_count"] = 1` (`reasons/api.py:462`).

That hardcoded `1` is intentional: after OR expansion, every generated SL justification has exactly one antecedent. If the code reported the original `len(antecedents)`, a node entered as `a,b,c` with `any_mode=True` would look like a 3-premise conjunctive ALL node to callers and to warning/hint logic. The tests assert the low count: `tests/test_any_mode.py:129` expects `premise_count == 1` for a 3-premise any-mode add. `tests/test_any_mode.py:198` verifies that any-mode nodes do not produce restoration-hint false alarms because each SL has one premise. The CLI warning also checks both count and mode: `_warn_multi_premise` only warns when `premise_count >= 3 and not any_mode` (`reasons/cli.py:35-38`).

Caveat: the observations show the guards, assignments, docstrings, and tests, but not the full loop bodies around `reasons/api.py:324` and `reasons/api.py:457`. The exact expansion loop is therefore inferred; the externally tested contract is clear.

## Topics to Explore

- [file] `reasons/api.py` — Read `add_node` and `add_justification` around lines 287-470 to see the exact expansion loops and `max_premises` calculation.
- [function] `reasons/api.py:add_justification` — Focus on the `any_mode` branch and the `premise_count` assignments at lines 462 and 469.
- [file] `reasons/network.py` — Inspect truth propagation and justification validity to confirm the node-level “IN if any justification is valid” rule.
- [file] `tests/test_any_mode.py` — Behavioral contract for OR expansion, retraction behavior, outlist copying, `premise_count`, and restoration hints.
- [general] `ALL-vs-ANY-classification-review` — Understand how review logic flags ALL/ANY misclassification without adding a special persisted justification type.

## Beliefs

- `node-in-any-justification-valid` — A node is IN if at least one of its justifications is valid, as documented in `reasons/__init__.py:30` and `reasons/network.py:963`.
- `any-mode-expands-or-sl-into-single-antecedent-justifications` — With `any_mode=True` and multiple antecedents, SL input is stored as one single-antecedent SL justification per antecedent, not as a new justification type.
- `premise-count-is-max-justification-premise-count` — The `add_node` API returns `premise_count` from `max_premises` (`reasons/api.py:400`), used for warnings and hints rather than total premise count across all justifications.
- `any-mode-add-justification-premise-count-is-one` — In the `add_justification` any-mode branch, `premise_count` is hardcoded to `1` because each generated justification has exactly one antecedent (`reasons/api.py:462`).
- `postgresql-rejects-any-mode` — Both `add_node` and `add_justification` raise `NotImplementedError` when `any_mode=True` on PostgreSQL (`reasons/api.py:313`, `reasons/api.py:432`).
