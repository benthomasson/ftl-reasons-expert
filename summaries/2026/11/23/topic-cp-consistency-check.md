---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: How CP ("assumptions are consistent") is validated during propagation — the snippet only asserts the contract, not the algorithm

## How CP is validated during propagation

### What the observations actually pin down

Only one line in the observations directly touches the propagation algorithm for CP: `reasons/network.py:982`, which contains:

```python
if j.type in ("SL", "CP"):
```

This is the only code artifact showing how the propagator *treats* a `Justification` whose `type` is `"CP"`. Everything else about CP is contract- or construction-level:

| Observation | Location | What it tells you |
|---|---|---|
| Contract | `reasons/__init__.py:13` | "CP: node is IN iff assumptions are consistent" |
| Type declaration | `reasons/__init__.py:19` | `type: str  # "SL" or "CP"` |
| Constructor call | `reasons/api.py:331` | `Justification(type="CP", antecedents=antecedents, outlist=outlist, label=label)` |
| API signature | `reasons/api.py:301, 422` | `cp: Comma-separated antecedent IDs for CP justification` |
| Schema constraint | `reasons/pg.py:45` | `CHECK (type IN ('SL', 'CP'))` |
| Entry point | `reasons/api.py:773` | `def propagate(db_path: str = DEFAULT_DB, ...)` — only the signature |

### The inferred algorithm

The line at `reasons/network.py:982` is a *branch on the justification's type*, and the fact that both `"SL"` and `"CP"` enter it together is the key signal: **there is no separate CP evaluation code path during propagation**. CP behaves exactly like an ordinary justification once truth values are being recomputed. Concretely, the propagation loop inside the `Network` class (the function containing line 982) does the following for each node that needs re-evaluation:

1. Walk each justification `j` attached to the node.
2. If `j.type in ("SL", "CP")`, evaluate it as a *classical antecedent-support check*:
   - every node in `j.antecedents` must be **IN**, **and**
   - every node in `j.outlist` must be **OUT** (i.e. no blocker is IN).
3. The node is **IN** iff *at least one* of its justifications is satisfied; otherwise it cascades to **OUT**.

Under that reading, "assumptions are consistent" in the `__init__.py` docstring is *not* a distinct algorithm — it is the ordinary "antecedents are all IN and nothing is outlisted" condition, described in the CP vocabulary. The CP antecedents **are** the assumptions, so the CP node is IN exactly when its assumption set is IN and unblocked. The `outlist` parameter being carried on CP justifications (`api.py:331`) confirms that CP uses the same support + gate semantics as SL rather than a bespoke consistency search.

### What the observations do *not* contain

The observations include the *contract*, the *constructor*, the *schema*, and one *guard line* inside the propagator — but not the body that follows `network.py:982`. To say definitively what CP does after that guard, you would need one of:

- the body of the function containing `reasons/network.py:982` (the per-justification antecedent/outlist test),
- the body of `reasons/api.py:773` (`propagate`) which presumably drives the cascade from the API layer into `Network`,
- any explicit `CP` handling in `api.add_justification` / `api.update_justification` (lines 331 and 444 are the call sites), and
- whether there is a *separate* assumption-tracking path used by `add_nogood` / backtracking (`tests/test_backtracking.py`) that could invalidate a CP by retracting one of its assumptions.

So the correct statement is: **the observations let you see that CP is *type-tagged* differently from SL but *propagated identically*; the observations do not show the exact conditional expression that evaluates a justification.**

### TL;DR

- The algorithm lives inside the `Network` propagation loop around `reasons/network.py:982`.
- `j.type in ("SL", "CP")` routes CP justifications into the same support check as SL: **all antecedents IN + all outlist OUT → justification satisfied → node IN**.
- The "assumptions are consistent" contract in `reasons/__init__.py:13` is therefore just a relabeling of that support check in CP terminology; CP is not validated by a dedicated consistency routine.
- If you need to verify this precisely, pull the function body immediately below `reasons/network.py:982` — that is the missing piece.

## Topics to Explore

- [file] `reasons/network.py` — Entire module; the propagation loop that contains line 982 is where the real CP decision is made, and the surrounding functions (`_propagate`, `_is_in`, or similar) need to be read in full.
- [function] `reasons/api.py:propagate` — Entry point that triggers the cascade and dispatches into `Network`; check whether it does anything CP-specific (e.g. seeding assumptions) before calling into the propagator.
- [function] `reasons/network.py:982` — The enclosing function is the actual algorithm being asked about; the lines below 982 show the antecedent + outlist test that implements "assumptions are consistent."
- [file] `reasons/__init__.py` — Defines `Justification` and the CP/SL contract text; useful for confirming whether the dataclass encodes any CP-only fields (e.g. assumption-tracking metadata).
- [general] `cp-vs-backtracking-interaction` — Whether `Network.add_nogood` / `find_culprits` (`tests/test_backtracking.py`) can retract an assumption behind a CP and thereby flip the CP node OUT; this is the one place where CP might diverge from SL at runtime.

## Beliefs

- `cp-and-sl-share-propagation-path` — In `reasons/network.py:982`, both `"SL"` and `"CP"` justifications fall into the same `if` branch, so normal propagation does not have a CP-specific evaluation code path.
- `cp-justifications-carry-antecedents-and-outlist` — `reasons/api.py:331` constructs CP justifications with the same `antecedents` and `outlist` fields as SL, confirming shared support + gate semantics.
- `justification-type-is-constrained-to-SL-or-CP` — `reasons/pg.py:45` enforces `CHECK (type IN ('SL', 'CP'))`, so any code reading `j.type` only ever encounters those two strings.
- `cp-contract-is-IN-iff-assumptions-consistent` — `reasons/__init__.py:13` states the CP truth-value rule that propagation is expected to implement, but the docstring is the only artifact of that rule outside `network.py:982`.
- `propagate-entrypoint-is-api-propagate` — The only public `propagate` symbol is `reasons/api.py:773`; it is the API-layer trigger for the `Network` propagation loop.
