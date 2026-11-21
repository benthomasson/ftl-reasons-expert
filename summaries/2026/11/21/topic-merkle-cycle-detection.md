---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How the verifier prevents infinite recursion on malformed antecedent graphs such as the artificial `a -> b -> a` cycle.

# How the Verifier Prevents Infinite Recursion on Cyclic Antecedent Graphs

## Short Answer

**The observations are insufficient to answer this question.** There is no cycle-prevention mechanism visible in the excerpted code, and the module that actually traverses antecedents is not shown.

## What the Observations Actually Show

### `reasons/verify.py` does not traverse graphs

The full content of `reasons/verify.py` (lines 1–137) contains exactly four functions, none of which walk antecedents or recurse:

| Line range | Function | Responsibility |
|---|---|---|
| ~49–67 | `read_source` | Resolves a path via `check_stale.resolve_source_path`, reads the file, truncates to `MAX_SOURCE_CHARS` |
| ~70–87 | `format_belief_for_verify` | Formats a single `(belief, source_content)` pair into prompt text |
| ~90–121 | `parse_verify_response` | Extracts and validates the LLM JSON response |
| ~124–137 | `verify_beliefs` | Joins formatted beliefs, calls `invoke_model`, returns parsed verdicts |

`verify_beliefs` requires the caller to hand it a pre-flattened list `beliefs_with_sources`. It has no notion of an antecedent, a parent, or a graph — so there is nothing here that could recurse.

### The traversal lives elsewhere

The tests reveal where the antecedent walk actually happens: `api.verify_belief(...)`.

- `TestApiVerifyBelief.test_trace_derived` (`tests/test_verify.py`, ~line 226) builds `d1` with `sl="p-a,p-b"` and then calls `api.verify_belief("d1", trace=True, db_path=db)`. The assertion is `len(result["beliefs_checked"]) == 2` and `set(result["verified"]) == {"p-a", "p-b"}` — i.e., the walk expands a single rooted node into its supporting premises.
- `TestApiVerifyBelief.test_derived_without_trace` (~line 254) shows `trace=False` checks only the derived node itself.

That means `verify_belief` (in `reasons/api.py`) — **which is not included in the observations** — is the function that must contain any visited-set, depth limit, or cycle check.

### The cycle guard is not in `verify.py`

The searches corroborate this:

- `search_cycle_guard_in_reasons` — pattern `cycle|cyclic|recursion|recursive|visited|seen|guard|stack` → **0 matches** across `reasons/`.
- `search_antecedent_traversal_in_verify` — pattern `antecedent|premise|parent|dependency|walk|traverse|...` in `verify.py` → **0 matches**.
- `search_cycle_tests` — pattern `cycle|...|a.*b.*a|malformed` → **0 matches**.

So there is neither a guard nor a regression test for `a -> b -> a` anywhere in the searched material.

### Two plausible (unverified) mechanisms

Given the evidence, a cycle guard in this codebase must live in one of two places, and the observations do not distinguish between them:

1. **At insertion time** — `api.add_node` accepts an `sl=` (support-link) parameter (used in `test_trace_derived` and `test_derived_without_trace`). If the graph layer rejects edges that would close a cycle, `a -> b -> a` could never exist in the DB in the first place. No test exercises this.
2. **At traversal time** — `api.verify_belief` could maintain a visited set while expanding `sl` links. `test_trace_derived` only exercises the happy path (`d1 -> {p-a, p-b}`), which has no cycle.

Neither is confirmed by the excerpt.

## What's Missing

To answer the original question you would need to inspect:

- `reasons/api.py` — specifically the implementation of `verify_belief`.
- The DB layer that backs `add_node` / `sl=` (possibly `reasons/db.py` or `reasons/store.py`) — to see whether a support-link insertion is validated against cycles.
- Any CLI-level tests in `tests/test_verify.py` beyond the derived-belief tracing cases — the deepest trace test only exercises a depth-1, acyclic graph.

Until one of those is read, the safest engineering statement is: **`reasons/verify.py` cannot prevent this cycle; preventing it is the responsibility of its caller, and no such guard is visible in the given excerpts.**

## Topics to Explore

- [function] `reasons/api.py:verify_belief` — Implements the `trace=True` antecedent expansion asserted by `test_trace_derived`; this is where a visited set or depth limit would live.
- [file] `reasons/api.py` — Contains the `add_node` implementation that consumes `sl=`; check whether it rejects support links that close a cycle at insert time.
- [file] `tests/test_verify.py` — Only contains acyclic tracing tests (`test_trace_derived`, `test_derived_without_trace`); a cycle guard would need a new `a -> b -> a` regression here.
- [general] `support-link-graph-invariants` — Whether `sl=` edges are validated (no self-loops, no cycles) at write time or only walked defensively at read time.
- [function] `reasons/verify.py:verify_beliefs` — The terminal call in the trace; note it receives a flat list, so any recursion boundary must be enforced by the caller before reaching it.

## Beliefs

- `verify-py-does-not-prevent-cycle-recursion` — `reasons/verify.py` contains no antecedent traversal, visited set, or recursion guard; cycle prevention must occur in a caller such as `reasons/api.py:verify_belief`.
- `trace-derived-belief-expands-to-supporting-premises` — `TestApiVerifyBelief.test_trace_derived` asserts that `api.verify_belief("d1", trace=True)` on a node built with `sl="p-a,p-b"` checks exactly `{"p-a", "p-b"}`.
- `trace-flag-toggles-premise-expansion` — `TestApiVerifyBelief.test_derived_without_trace` demonstrates that `trace=False` checks only the requested node (`d1`), leaving antecedent expansion off.
- `derived-node-tracing-requires-sl-parameter` — `test_trace_derived` and `test_derived_without_trace` both create their derived node via `api.add_node(..., sl=...)`, confirming `sl` is the field that encodes a node's antecedent edges.
- `cycle-guard-search-in-reasons-returns-empty` — A repo-wide search of `reasons/` for `cycle|cyclic|recursion|recursive|visited|seen|guard|stack` returns zero matches, so no cycle guard is documented inline.
