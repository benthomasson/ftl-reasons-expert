---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Assumptions such as acyclicity, valid `supporting_justification` indexes, and missing antecedent handling.

## Assumptions in the observed code

### 1. `supporting_justification` is a nullable index into `justifications`

The data model explicitly defines:

- `reasons/__init__.py:37` — `supporting_justification: int | None = None  # index into justifications list`

So the intended contract is:

- `None` means “no supporting justification selected”
- a non-`None` integer is supposed to be a valid index into `node.justifications`

Storage mirrors this assumption:

- `reasons/storage.py:22` — `supporting_justification INTEGER DEFAULT NULL`
- `reasons/storage.py:148-149` adds the column if missing on an older schema
- `reasons/storage.py:235-243` persists `node.supporting_justification`
- `reasons/storage.py:417` reconstructs a node with `supporting_justification=supporting_j`

The observed schema does not show a foreign-key or `CHECK` constraint tying this integer to `len(justifications)`. That means index validity is likely enforced, if at all, at the application layer.

The clearest dereference is in `reasons/merkle.py:28-31`:

```python
if not node.justifications or node.supporting_justification is None:
    ...
j = node.justifications[node.supporting_justification]
```

This code explicitly handles two cases:

- no justifications
- no selected supporting justification

But it assumes that if `supporting_justification` is not `None` and `justifications` is non-empty, then the integer is in range. No bounds check is visible in the observation. An invalid stored value, a stale value after mutating `justifications`, or a bug in index assignment could therefore cause an `IndexError` at this dereference.

Other sites read the field without demonstrating validation:

- `reasons/api.py:3379` — `si = node.supporting_justification`
- `reasons/derive.py:126` — `si = node_data.get("supporting_justification")`
- `reasons/network.py:853` — `si = node.supporting_justification`

`network.py` also mutates the field:

- `reasons/network.py:685` — `node.supporting_justification = None`
- `reasons/network.py:971` — `node.supporting_justification = i`
- `reasons/network.py:973` — `node.supporting_justification = None`

So the code assumes that whatever writes the index also preserves the invariant `0 <= si < len(node.justifications)` whenever `si is not None`.

### 2. Acyclicity is not obviously a global schema invariant, but recursive paths are guarded

Antecedents are free-form node IDs:

- `reasons/__init__.py:20` — `antecedents: list[str] = field(default_factory=list)  # inlist: must be IN`

There is no observed validation in the data model that prevents cycles or dangling references. The existence of explicit cycle guards suggests the code anticipates cycles in at least some recursive evaluation paths:

- `reasons/api.py:3378` — `memo[nid] = 0  # cycle guard`
- `reasons/derive.py:123` — `memo[node_id] = 0  # cycle guard`

These guards look like DFS “currently visiting” markers. They likely prevent infinite recursion by treating a revisited in-progress node as a default/false value, rather than by raising a validation error. That is an important distinction: cycle safety in these paths does not necessarily mean cycles are semantically valid or rejected elsewhere.

There is also a targeted test:

- `tests/test_merkle.py:193-200` — `test_cycle_safety`
- It artificially creates a cycle and verifies that verification does not infinite-loop.

So the observed evidence supports this interpretation:

- The graph may contain cycles in practice or in malformed data.
- Some recursive algorithms guard against them.
- The observations do not prove that all graph operations are cycle-safe or that cycles are globally forbidden.

### 3. Missing antecedent handling is not established by the observations

The search for handling of missing/not-found/invalid antecedent cases returned no matches:

- `find_missing_antecedent_handling.total_count: 0`

However, that result is suspicious because patterns like `None` appear elsewhere in the supplied observations. The search may be scoped differently or incomplete. Therefore, the absence of matches should not be treated as proof that no missing-antecedent handling exists.

What the observations do show is:

- Antecedents are stored as strings in `Justification.antecedents` (`reasons/__init__.py:20`).
- No validation of antecedent existence is visible in the model.
- Some recursive paths use `memo` cycle guards (`reasons/api.py:3378`, `reasons/derive.py:123`), but those are cycle guards, not necessarily missing-reference handlers.
- Some readers use `.get()` for `supporting_justification`, e.g. `reasons/derive.py:126` and `reasons/build_wiki.py:109` / `reasons/build_wiki.py:200`, which tolerates a missing key in a dictionary. That does not imply missing antecedent node IDs are handled.

So the safe conclusion is:

- The observed evidence is insufficient to describe the exact missing-antecedent behavior.
- We cannot tell whether a dangling antecedent causes a `KeyError`, is skipped, is treated as inactive/false, or is validated earlier in the API/CLI/network layer.
- The relevant functions around `reasons/api.py:3378`, `reasons/derive.py:123`, and `reasons/merkle.py:28` need to be inspected directly.

## Evidence gaps

The observations are mostly line-level matches, not full function bodies. To fully answer this question we would need to inspect:

- The recursive evaluator around `reasons/api.py:3378-3379`
- The recursive derivation logic around `reasons/derive.py:123-126`
- The Merkle verification logic around `reasons/merkle.py:28-31`
- The code that assigns `supporting_justification` in `reasons/network.py:971`
- Any validation or normalization path when nodes are added or when justifications are mutated

## Topics to Explore

- [file] `reasons/merkle.py` — Shows the clearest `supporting_justification` dereference and the observed cycle-safe verification behavior.
- [file] `reasons/derive.py` — Contains a cycle guard and reads `supporting_justification`; inspect how antecedent lookups behave when IDs are missing.
- [file] `reasons/api.py` — Contains another cycle guard and several serialization/read paths for `supporting_justification`.
- [file] `reasons/network.py` — Mutates `supporting_justification`; determine whether index validity is enforced when assigning `i`.
- [general] `missing-antecedent-handling` — Search and inspect resolver/evaluator paths to confirm whether dangling antecedent IDs are rejected, ignored, or treated as inactive.

## Beliefs

- `supporting-justification-is-a-nullable-index` — `supporting_justification` is typed as `int | None` and stored as a nullable integer, where `None` means no selected justification.
- `supporting-justification-index-has-no-observed-bounds-check-at-dereference` — `reasons/merkle.py:28-31` checks for `None` and empty justifications but then indexes directly, so in-range index validity is an application-level assumption.
- `cycle-guards-prevent-infinite-recursion-in-some-evaluators` — `reasons/api.py:3378` and `reasons/derive.py:123` set memo entries as cycle guards during recursive evaluation.
- `cycle-safety-is-tested-for-merkle-verification` — `tests/test_merkle.py:193` constructs an artificial antecedent cycle and verifies that verification does not infinite-loop.
- `missing-antecedent-handling-is-not-established-by-the-gathered-evidence` — The missing-reference search returned no matches, and the visible snippets do not show how dangling antecedent IDs are handled.
