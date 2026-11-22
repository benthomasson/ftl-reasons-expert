---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Determine whether the existing nogood/contradiction machinery can supply the CP “assumptions are consistent” predicate.

## Verdict

No — the existing nogood/contradiction machinery does **not** currently supply the CP “assumptions are consistent” predicate as an implemented API. It provides the storage model and mutation/import/export paths for nogoods, plus an LLM-based contradiction **detector**, but there is no observed function that takes a set of assumptions and returns whether they are consistent.

If assumptions are represented as node IDs, the nogood store is the right substrate for a conservative predicate:

```text
are_assumptions_consistent(S) :=
    no stored nogood ng such that ng.nodes ⊆ S
```

But that function is not present in the observations. It would need to be added, likely on `Network` or as an API helper. Its meaning would be “consistent with recorded nogoods,” not necessarily fully logically consistent.

## Evidence

- CP semantics are only documented at `reasons/__init__.py:13`:  
  `- CP (Conditional Proof): node is IN iff assumptions are consistent`  
  The consistency-predicate grep found no `is_consistent`, `check_consist`, or `def .*consist` functions, and the only assumptions/consistency match was that docstring.

- `Nogood` exists at `reasons/__init__.py:53`, and `net.nogoods` is exported at `reasons/api.py:1879-1881`. So the system stores nogoods as data.

- `api.add_nogood` at `reasons/api.py:1759` records a nogood and returns `nogood_id`, `nodes`, `changed`, and `backtracked_to` at `reasons/api.py:1763`. That is mutation/backtracking, not a query predicate. The `changed`/`backtracked_to` fields suggest internal backtracking logic may exist, but the observations do not show an exposed “check assumptions” function.

- Nogoods are sets of node IDs. `tests/test_nogood_id.py` verifies IDs like `nogood-a-b` are derived from sorted node IDs, duplicate nogoods get suffixes, and IDs persist/import correctly. This supports a subset-based consistency check if assumptions are also node IDs.

- `reasons/contradictions.py` is a discovery pipeline, not a runtime consistency predicate. `detect_contradictions(...)` filters to current `"IN"` beliefs, batches them, asks an LLM for contradictions, and parses `### NOGOOD ...` proposals into dicts with `id`, `claims`, `analysis`, and `severity`. `detect_contradictions_semantic(...)` does the same after semantic clustering. It can propose nogoods to add, but it cannot certify consistency: it is LLM-based, batch-limited, and only checks currently held beliefs.

- Tests cover contradiction parsing/batching in `tests/test_contradictions.py` and nogood ID persistence/import in `tests/test_nogood_id.py`, but there is no test for a CP assumption-consistency predicate.

## What Would Be Needed

To supply the CP predicate, add a deterministic query such as:

- `Network.are_assumptions_consistent(assumption_ids: set[str]) -> bool`
- or `api.check_assumptions_consistent(assumption_ids, db_path=...)`

It should iterate stored `net.nogoods` and check whether any nogood is a subset of the assumption IDs. It should also return the violating nogood ID(s) for diagnostics.

Missing from the observations: the actual implementation of `Network.add_nogood`, the `Nogood` class fields, the database schema, and the CP evaluation call site. Those are needed to confirm whether internal backtracking logic already performs a similar subset check and whether assumptions are indeed represented as node-ID sets.

## Topics to Explore

- [file] `reasons/network.py` — Inspect `Network.add_nogood`, `net.nogoods`, and propagation/backtracking logic to see whether any subset-check logic already exists.
- [function] `reasons/api.py:add_nogood` — Understand exactly how nogoods are recorded and what `changed` and `backtracked_to` mean.
- [file] `reasons/__init__.py` — Inspect the `Nogood` class and CP documentation for intended data model and semantics.
- [general] `cp-assumption-consistency` — Search for where CP nodes are evaluated and whether a consistency predicate is called or missing.
- [file] `reasons/contradictions.py` — Understand the LLM-based detection pipeline and why it is not a complete runtime consistency check.

## Beliefs

- `no-consistency-predicate-exists` — No function definition matching `is_consistent`, `check_consist`, or `def .*consist` was found in the observed codebase.
- `nogoods-are-node-id-sets` — Nogood IDs are derived from sorted node IDs and persisted/imported as sets of node IDs.
- `add-nogood-mutates-and-backtracks` — `api.add_nogood` records a nogood via `net.add_nogood(node_ids)` and returns `changed` and `backtracked_to`.
- `contradiction-detection-filters-to-in-nodes` — `detect_contradictions` only checks nodes with `truth_value == "IN"` and returns LLM-proposed nogood dictionaries.
- `cp-consistency-documented-not-implemented` — The only observed assumptions/consistency match is the CP docstring at `reasons/__init__.py:13`.
