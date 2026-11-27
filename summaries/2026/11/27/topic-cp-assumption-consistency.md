---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search for where CP nodes are evaluated and whether a consistency predicate is called or missing.

## Search for CP node evaluation and consistency predicate usage

### Short answer

CP nodes **are** dispatched for truth-value evaluation in `reasons/network.py` at line **982** inside `_compute_truth` (the method invoked from `add_node`, `add_justification`, and `supersede`). However, **no dedicated consistency predicate is called anywhere in the code we can see**. The documentation (`reasons/__init__.py:13`) asserts "CP node is IN iff assumptions are consistent", but the token-grep for `predicate`, `is_consistent`, `check_consistency`, and `CP.*(eval|check|verify|consistent)` produced **zero real code matches** — only the doc string and unrelated wording.

Whether this is *actually* missing or is inlined below line 500 of `network.py` cannot be confirmed from the observations alone.

### Where CP nodes are evaluated

- **`reasons/network.py:982`** — inside `_compute_truth` (inferred from surrounding code):
  ```python
  if j.type in ("SL", "CP"):
  ```
  This is the only place the `"CP"` type is branched on during truth-value computation. `_compute_truth` is called from `Network.add_node` (when justifications are present), from `Network.supersede`, and again whenever a dependent is re-evaluated during propagation.

- **`reasons/api.py:331` and `reasons/api.py:444`** — CP justifications are constructed with `type="CP"` via the API layer (`jtype = "CP"`). These are the entry points that create CP-typed `Justification` objects, which then enter `_compute_truth`.

- **`reasons/pg.py:3683`** — the Postgres store also writes a justification with `"type": "CP"` when persisting, so CP nodes flow through both the in-memory network and the persistence layer.

- **`reasons/derive.py`** — *not* involved in CP evaluation. It is the LLM "derive deeper conclusions" module (`DERIVE_PROMPT`, `parse_proposals`); its `outlist`-gated GATE proposals are separate from `Justification.type == "CP"`.

### Whether a consistency predicate is called

Searches that would reveal a consistency predicate:

| Search | Result |
|---|---|
| `predicate` | 0 matches (no function by that name) |
| `is_consistent` / `check_consistency` | 0 matches |
| `consistency\|consistent` | 10 matches — **all docstrings, comments, or unrelated words** (`metadata.py:3`, `cluster.py:50`, `derive.py:38`, `api.py:3921` phrase list, etc.) |
| `CP.*(eval\|check\|verify\|valid\|consistent)` | 1 match — the `reasons/__init__.py:13` doc string only |

The closest thing to a consistency check is **`Network.verify_dependents` at `reasons/network.py:47`**, but its docstring explicitly says it checks whether the `dependents` reverse index matches what justifications imply — it is *index* consistency, not *CP assumption* consistency.

`Network.trace_assumptions` (around `reasons/network.py` mid-file) walks back through justifications to enumerate premises. That is the natural input a CP consistency check would consume, but it is only an enumeration — it does not evaluate anything.

### What's missing from the observations

`read_network_file` returned only **500 of 1002 lines**, and line **982** sits beyond that cutoff. The full body of `_compute_truth` around 960–1002 was not observed. So we can say:

- **Confirmed:** CP is type-dispatched at `network.py:982`.
- **Confirmed:** No named predicate (`is_consistent`, `check_consistency`, `predicate`) exists anywhere we saw.
- **Unconfirmed:** Whether `_compute_truth` inlines consistency logic for CP (e.g. recursively checking that assumptions behind a CP justification are all `IN`), or treats CP identically to SL. The single `if j.type in ("SL", "CP"):` line strongly suggests a **uniform treatment**, but the body that follows was not read.

A reasonable hypothesis from these fragments: the code documents CP semantics in `reasons/__init__.py:13` but the implementation may evaluate CP justifications the same way as SL, leaving the "consistent assumptions" requirement unenforced. That would be the bug the search was meant to confirm — but the 500-line truncation means we should not declare it a confirmed gap.

### Recommended next read

Read `reasons/network.py` lines **900–1002** (or grep for `def _compute_truth` and read 80 lines forward) to see the full branch after line 982. The single `if j.type in ("SL", "CP"):` is either:

1. The start of a `for j in node.justifications:` loop that treats both types identically → CP consistency is **missing**, or
2. The wrapper around a CP-specific sub-branch that calls a helper (which may not be named "consistency") → CP consistency is **inlined**.

## Topics to Explore

- [file] `reasons/network.py` — Read lines 900–1002 to see the full `_compute_truth` body and determine how the CP branch at line 982 resolves truth.
- [function] `reasons/network.py:_compute_truth` — The actual CP/SL dispatch; confirm whether CP assumptions are checked for all-IN consistency or treated like SL.
- [function] `reasons/network.py:trace_assumptions` — Enumerates premise IDs; this is the natural input a CP consistency check would consume, and its absence of callers around line 982 would confirm a gap.
- [file] `reasons/api.py` — Lines 301–331 and 422–444 show how CP justifications are constructed (`jtype = "CP"`), which is where a consistency precondition would most naturally be enforced.
- [general] `cp-consistency-semantics` — Investigate Doyle/ATMS CP semantics ("assumption-based" vs "conditioned" justifications) to know what "consistency" should mean in this codebase versus what is implemented.

## Beliefs

- `cp-justifications-are-type-dispatched-in-compute-truth` — CP nodes reach truth-value evaluation at `reasons/network.py:982` via `_compute_truth`, which branches on `j.type in ("SL", "CP")`.
- `no-dedicated-consistency-predicate-exists` — No symbol named `is_consistent`, `check_consistency`, or `predicate` appears anywhere in the searched codebase (`find_predicate_usage` = 0 matches).
- `cp-doc-claims-consistency-requirement-not-backed-by-named-call` — `reasons/__init__.py:13` documents CP as "IN iff assumptions are consistent" but no corresponding function call is visible in the 500-line `network.py` excerpt.
- `verify-dependents-checks-index-not-assumptions` — `Network.verify_dependents` (`reasons/network.py:47`) only validates the `dependents` reverse index and is unrelated to CP assumption consistency.
- `cp-justifications-are-created-in-api-and-persisted-via-pg` — CP-typed justifications originate from `reasons/api.py:331` / `reasons/api.py:444` and are round-tripped through `reasons/pg.py:3683`.
