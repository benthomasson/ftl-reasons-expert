---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Model `SL` rules as defaults and nogoods as integrity constraints to reason about restoration and retraction.

# Modeling `SL` rules as defaults and nogoods as integrity constraints

## SL justifications are defeasible defaults

An `SL` justification is the system’s default-inference rule: a node is normally `IN` when its support list / antecedents are `IN` (`reasons/__init__.py:11`). The justification also carries an `outlist`, which acts as an exception/gate: if a blocker in the outlist is `IN`, the default is defeated even when the positive antecedents hold. This is visible in the `Justification(type="SL", antecedents=[...], outlist=outlist, ...)` construction in `reasons/api.py:326-334`.

Multiple `SL` justifications are alternative supports. `any_mode` is especially important: it expands one multi-antecedent `SL` justification into one justification per antecedent (`reasons/api.py:306`, `reasons/api.py:457-460`). In other words, normal `SL` is an AND-default over antecedents; `any_mode` turns it into an OR-default.

The prompt in `reasons/derive.py` describes the intended mental model directly: an outlist-gated conclusion is `OUT` while the outlist node is `IN`, and flips `IN` when the blocker goes `OUT`. The restoration example in `reasons/api.py:1428-1429` matches this: challenging a scope belief can make a defeater go `OUT`, restoring the original belief.

## Nogoods are hard consistency constraints

A nogood is a constraint over a set of node IDs: those nodes must not all be `IN` simultaneously. `add_nogood` accepts `node_ids` and returns `nogood_id`, `nodes`, `changed`, and `backtracked_to` (`reasons/api.py:1759-1781`). The presence of `backtracked_to` shows that enforcing a nogood is not just a boolean check; it can roll back or retract supporting state to restore consistency.

The `Nogood` class is defined in `reasons/__init__.py:53`. Nogoods are exposed in the network snapshot (`reasons/api.py:1879-1881`) and can be imported from Markdown/JSON (`reasons/api.py:1904-1921`).

A useful distinction: `check_integrity` in `reasons/api.py:2731` and the CLI `check-integrity` command in `reasons/cli.py:3525` are Merkle-hash integrity checks for text mutation detection, not logical nogood consistency checks. The Merkle module docstring confirms this at `reasons/merkle.py:1`.

## How retraction and restoration follow from the model

Treat the belief network as a non-monotonic state machine:

1. **Default closure:** apply `SL` rules to determine what should be `IN`, subject to outlist blockers.
2. **Integrity check:** enforce nogoods. If a nogood is violated, retract/backtrack some node(s).
3. **Recompute closure:** retracting a node can cascade `OUT` to its dependents; retracting a blocker can restore gated beliefs to `IN`.
4. **Iterate to a fixpoint:** the final result reports both directions of change.

The API reflects this directly:

- `retract_node` calls `net.retract(...)` and returns `changed`, `went_out`, `went_in`, and `restoration_hints` (`reasons/api.py:473-509`).
- `what_if_retract` performs the same operation in memory and separates affected nodes into `retracted` (`went OUT`) and `restored` (`went IN`), sorting by depth (`reasons/api.py:512-576`).
- The comment at `reasons/api.py:519` explicitly ties restoration to outlist: “restoration from outlist — gated beliefs whose blocker is removed.”
- The simulate/assert block around `reasons/api.py:582-643` tracks both restoration cascades (`went IN`) and retraction cascades (`went OUT`).

So the key reasoning rule is:

> A retraction is a repair action. A restoration is a consequence of removing a blocker or restoring support in a default rule. Nogoods force the system to choose a repair when defaults would otherwise leave an inconsistent set `IN`.

## What the observations do not show

The snippets show the API contracts and construction of `SL` justifications, but not the core enforcement algorithm. Missing details include:

- The full `Nogood` class fields and parsing format.
- The implementation of `net.add_nogood(...)` and `net.retract(...)`.
- How `backtracked_to` is selected — chronological backtracking, minimal retraction, priority ordering, etc.
- The exact fixpoint algorithm and how `restoration_hints` are computed.
- How multiple nogoods interact and whether nogood violations are minimized.

So the model above is justified by the observable API and comments, but the precise nogood-repair policy cannot be proven from these observations alone.

## Topics to Explore

- [function] `reasons/api.py:add_nogood` — Inspect how nogood constraint enforcement chooses what to retract and how `backtracked_to` is computed.
- [function] `reasons/api.py:retract_node` — Trace how `net.retract(...)` cascades `went_out` and computes `went_in`/`restoration_hints`.
- [function] `reasons/api.py:what_if_retract` — Study the read-only simulation of retraction and restoration, especially outlist-gated beliefs.
- [file] `reasons/__init__.py` — Review the `Nogood` class and the network primitives for truth evaluation and justification activation.
- [general] `nogood-backtracking-policy` — Determine whether nogood enforcement is minimal, chronological, priority-based, or otherwise.

## Beliefs

- `sl-justification-is-default-rule` — An `SL` justification stores antecedents and an outlist, and the network treats it as a defeasible support rule (`reasons/__init__.py:11`, `reasons/api.py:326-334`).
- `any-mode-expands-and-to-or` — `any_mode` splits a multi-antecedent `SL` justification into one justification per antecedent, changing AND support into OR support (`reasons/api.py:306`, `reasons/api.py:457-460`).
- `add-nogood-contract-includes-backtracking` — `add_nogood` returns `nogood_id`, `nodes`, `changed`, and `backtracked_to`, so nogood enforcement can report backtracking (`reasons/api.py:1759-1781`).
- `retract-node-reports-both-retraction-and-restoration` — `retract_node` returns `changed`, `went_out`, `went_in`, and `restoration_hints` (`reasons/api.py:482-509`).
- `check-integrity-is-merkle-verification` — `check_integrity` and the CLI `check-integrity` command verify Merkle hashes for text mutation detection, not logical nogood satisfaction (`reasons/api.py:2731`, `reasons/cli.py:3525`, `reasons/merkle.py:1`).
