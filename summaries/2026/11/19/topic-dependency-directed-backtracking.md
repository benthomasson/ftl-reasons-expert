---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Explains `_find_culprits_internal` + `_entrenchment`; the retraction policy is a product decision, not just an implementation detail.

## What these two functions actually are

They are the two halves of a single decision that the system makes every time a contradiction (a *nogood*) is found: **"which held belief do we give up?"**

- `_find_culprits_internal(self, cur, nogood_node_ids)` — `reasons/pg.py:3287`. Given the set of node IDs that cannot all be IN at once, it produces a candidate set of *blameable* nodes ("culprits") whose removal would resolve the conflict.
- `_entrenchment(self, cur, node_id)` — `reasons/pg.py:3328` (Postgres) and `reasons/network.py:291` (SQLite/in-memory). Returns an `int` score: how strongly the network is committed to a node.

The connective tissue is visible in the line numbers themselves: `_find_culprits_internal` is defined at 3287 and `_entrenchment` at 3328, so **line 3316 — `entrenchment = self._entrenchment(cur, premise_id)` — is inside `_find_culprits_internal`'s body.** The blame-finder asks the entrenchment scorer for a ranking. Note the loop variable name: `premise_id`, not `node_id`. The culprits are drawn from *premises*, not from the nogood's members, which is the correct TMS behaviour — retracting an intermediate derived node while its premise stays IN just re-derives it on the next propagation pass.

## The pipeline these sit in

1. `reasons/contradictions.py` proposes nogoods. `detect_contradictions` scans only IN beliefs (`node.get("truth_value") == "IN"`), and `parse_contradiction_response` enforces `len(claims) >= 2` before returning a contradiction. This module decides *that* a conflict exists — it does not decide blame.
2. The nogood is recorded (`SKILL.md:99`, `reasons nogood node-a node-b` — "Uses dependency-directed backtracking to trace…").
3. `_find_culprits_internal` does the trace, calling `_entrenchment` per premise.
4. The chosen node goes OUT via `net.retract(...)` (`reasons/api.py:473–489`), cascading to dependents, with `what_if_retract` (`reasons/api.py:512`) offering a read-only simulation that returns `retracted`/`restored` grouped by depth.

The two call sites at `reasons/pg.py:2084` and `2122` both receive `node_ids`, consistent with a nogood member set being handed to the trace.

## Why the retraction policy is a product decision

Dependency-directed backtracking tells you which *sets* of premises are sufficient to explain the contradiction. It does **not** tell you which of several valid resolutions to take. That choice is `_entrenchment`, and the tests in `tests/test_backtracking.py` pin it as policy:

| Assertion | Line | The product rule it encodes |
|---|---|---|
| `_entrenchment("evidence") > _entrenchment("basis")` | 230 | Observed evidence outranks assumed foundations — we'd rather abandon a premise we made up than a thing we saw. |
| `_entrenchment("sourced") > _entrenchment("unsourced")` | 259 | Provenance quality raises entrenchment; poorly-sourced beliefs are the first to go. |
| `_entrenchment("obs") > _entrenchment("der")` | 266 | Observed beats derived — retract the inference, keep the observation. |
| `_entrenchment("popular") > _entrenchment("lonely")` | 275 | Widely-depended-on beliefs are expensive to retract (each retraction cascades); isolated ones are cheap. |

Every one of those inequalities is a falsifiable statement about how much the product is willing to churn a user's belief set, and each maps directly to user-visible output: which rows appear under `retracted` in a `what_if_retract` response (`reasons/api.py:551–576`), what `reasons list --status OUT` shows, and what the audit trail in `reasons log` records. Change a weight and you change what the tool *concludes*, not merely how fast it runs. That is why these belong in tests, not just in code.

## What the observations do not give us

Be explicit with your colleague about the gap:

- **No bodies.** Neither function's source is in the observations, so I cannot state the scoring formula, the tie-break rule between equally-unentrenched premises, the return scale of `_entrenchment`, or which node fields (`type`, `source`, dependent count, depth) feed the score. The four tests constrain the ordering but not the arithmetic.
- **No call-site context** for `reasons/pg.py:2084` / `2122` — we know they pass `node_ids`, not which operations they serve.
- **Backend parity is asserted, not shown.** Two `_entrenchment` implementations exist (`reasons/network.py:291`, `reasons/pg.py:3328`). The tests at `test_backtracking.py:230–275` exercise `net._entrenchment`, i.e. the `network.py` path only; nothing in the observations proves the Postgres version returns the same ordering.
- **Metadata inconsistency worth flagging:** `search_retraction_policy_tests` returned `total_count: 0`, yet `find_entrenchment_usages` lists five `_entrenchment` references in `tests/test_backtracking.py`. Treat the empty search as a tooling/glob artifact, not as evidence the policy is untested.

## Topics to Explore

- [file] `reasons/pg.py` — Read lines 3287–3330 verbatim to get the actual entrenchment formula and the culprit-selection loop; this is the only place the policy is executed.
- [function] `reasons/network.py:_entrenchment` — The reference scoring implementation for the SQLite backend; compare it against the Postgres version for parity.
- [file] `tests/test_backtracking.py` — The product policy, expressed as ordering assertions (lines 230, 259, 266, 275); this is the contract any weight change must preserve.
- [file] `reasons/contradictions.py` — Upstream producer of nogoods; explains what shape of input `_find_culprits_internal` receives.
- [general] `dual-backend-parity` — How the two `_entrenchment` implementations are kept in sync, and whether any test asserts the Postgres path's ordering.

## Beliefs

- `find-culprits-internal-consumes-entrenchment` — `reasons/pg.py:3316` calls `self._entrenchment(cur, premise_id)` from within the body that begins at `reasons/pg.py:3287`, so culprit selection is ranked by entrenchment score.
- `culprit-selection-operates-on-premises` — The iteration variable at `reasons/pg.py:3316` is `premise_id`, indicating culprits are chosen among support nodes rather than among the nogood's own members.
- `entrenchment-is-implemented-per-backend` — `_entrenchment` has two independent definitions, `reasons/network.py:291` and `reasons/pg.py:3328`, with different signatures.
- `entrenchment-ranks-observed-above-derived` — `tests/test_backtracking.py:266` asserts `net._entrenchment("obs") > net._entrenchment("der")`.
- `entrenchment-ranks-sourced-above-unsourced` — `tests/test_backtracking.py:259` asserts `net._entrenchment("sourced") > net._entrenchment("unsourced")`.
