---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Decide whether cycle detection should raise, return a sentinel, or emit structural findings consistently across verification and backfill paths.

## Decision

Emit **structural findings** consistently for cycle detection, rather than making cycle detection primarily raise or returning a bare sentinel.

Concretely:

- **Verification** should continue to return a result object containing `findings`, with a canonical cycle finding type such as `"cycle"` / `"cyclic_antecedents"`.
- **Backfill** should run the same graph traversal/cycle detection and include the same findings in its result, rather than silently computing hashes through a cyclic graph.
- **The `memo[nid] = 0` guard** seen at `reasons/api.py:3378` and `reasons/derive.py:123` can remain an internal recursion guard, but it should not be the public contract. If traversal hits a cycle, it should record a structural finding. A silent `0` sentinel risks producing a misleading hash or derivation result.
- **Raising** should be reserved for explicit strict mode or programmer-error cases. Default CLI/backfill paths are batch operations and should report findings, not abort the whole operation halfway through.

## What the observations show

The codebase already has a partial “structural findings” contract:

- `reasons/merkle.py:112` initializes `findings = []`.
- `reasons/merkle.py:118` and `reasons/merkle.py:124` append findings.
- `reasons/api.py:2739-2740` filters verification results by finding type:
  ```python
  text_mutations = [f for f in result["findings"] if f["type"] == "text_mutation"]
  chain_mutations = [f for f in result["findings"] if f["type"] == "chain_mutation"]
  ```
  That strongly suggests callers expect non-raising structural findings from Merkle verification.

However, cycle handling is not clearly part of that contract:

- `reasons/api.py:3378` has `memo[nid] = 0  # cycle guard`.
- `reasons/derive.py:123` has the same `memo[node_id] = 0  # cycle guard`.
- These look like internal recursion guards, not explicit cycle detectors.
- `tests/test_merkle.py:193-200` only asserts that verification “doesn’t infinite-loop on hypothetical cycle in antecedents.” The test name is `test_cycle_safety`, and the docstring is about safety, not about a defined error/finding contract. There is no observed assertion that a cycle produces a specific finding or exception.
- There is no `detect_cycle` or `find_cycle` symbol in the observations.

The backfill path also does not visibly share whatever cycle-handling contract verification has:

- `reasons/api.py:2748-2752` defines `backfill_hashes` and delegates to `reasons.merkle.backfill_hashes`.
- `reasons/merkle.py:130` defines the actual `backfill_hashes(net)`.
- `reasons/cli.py:1183-1185` wires `cmd_backfill_hashes` to `api.backfill_hashes`.
- `reasons/cli.py:1105-1112` formats backfill output using counts like `backfilled` and `rehashed`, not structural findings.

So the current picture is:

- Verification path: has a `findings` list and structural finding types.
- Cycle guards: exist as `memo[...] = 0`, but their observable contract is unclear.
- Backfill path: recomputes hashes and reports counts, but no observed cycle-finding channel.
- Tests: prove no infinite loop, but do not prove consistent error semantics.

## Recommended consistency rule

Use this rule across verification and backfill:

1. A cycle in antecedent/justification structure is a **data-integrity finding**.
2. Both verification and backfill should return it in the same `findings` shape.
3. The finding should include at least:
   - `type`: e.g. `"cycle"` or `"cyclic_antecedents"`
   - the node ids participating in the cycle
   - a message explaining that traversal was truncated
4. Traversal should still terminate safely, e.g. by using the `memo` guard internally.
5. Backfill should not silently produce hashes for cyclic regions without surfacing the cycle.
6. Strict mode may raise, but default verification and CLI/backfill paths should report findings.

Example finding shape, consistent with the existing `result["findings"]` pattern:

```python
{
    "type": "cycle",
    "nodes": ["a", "b"],
    "message": "Cycle detected in antecedent graph; traversal truncated",
}
```

This keeps the existing verification API shape intact while making backfill expose the same integrity problem.

## Caveat

The observations are not enough to know exactly how `reasons/merkle.py:112-130` builds and returns findings, or what function contains the `memo[nid] = 0` guard at `reasons/api.py:3378`. To finalize the implementation, inspect the full bodies around `reasons/merkle.py:100-160`, `reasons/api.py:3340-3395`, and the assertions in `tests/test_merkle.py:193-205`.

## Topics to Explore

- [file] `reasons/merkle.py` — Inspect the `findings` construction, verification logic, and `backfill_hashes` implementation to see where cycle findings should be added.
- [file] `reasons/api.py` — Compare the verification result filtering at `2739-2740` with the backfill wrapper at `2748-2752` and the cycle guard at `3378`.
- [file] `tests/test_merkle.py` — Check whether `test_cycle_safety` asserts only non-infinite-loop behavior or also asserts a finding/exception contract.
- [general] `cycle-finding-schema` — Define the canonical finding type and fields for cyclic antecedent graphs.
- [general] `strict-vs-reporting-modes` — Decide whether verification/backfill need an optional strict mode that raises instead of reporting.

## Beliefs

- `verification-exposes-structural-findings` — The Merkle verification path builds a `findings` list and `reasons/api.py:2739-2740` filters it by `text_mutation` and `chain_mutation`.
- `backfill-delegates-to-merkle` — `reasons/api.py:backfill_hashes` delegates to `reasons.merkle.backfill_hashes` and returns its result.
- `cycle-guard-is-not-a-public-detector` — `reasons/api.py:3378` and `reasons/derive.py:123` use `memo[...] = 0` as an internal cycle guard rather than an explicit raise or named finding.
- `cycle-test-only-proves-termination` — `tests/test_merkle.py:193-200` documents cycle safety as avoiding an infinite loop, not as a defined exception or finding contract.
- `backfill-cli-reports-counts-not-findings` — `reasons/cli.py:1105-1112` formats backfill output as `backfilled`/`rehashed` counts rather than structural findings.
