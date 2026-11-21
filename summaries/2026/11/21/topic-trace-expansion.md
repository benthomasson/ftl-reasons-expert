---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `sl` premises are recursively collected and de-duplicated for `trace=True`, and how `is_derived` is computed.

## How `sl` premises and `is_derived` are implemented, per the observations

### `is_derived`

The observations show that `is_derived` is part of the API result shape and is set from a local `has_justifications` value:

- `reasons/api.py:3805` declares the field as a boolean in a dict/type shape:
  ```python
  "is_derived": bool, "dry_run": bool}
  ```

- Both observed API result construction sites assign:
  ```python
  "is_derived": has_justifications,
  ```
  at `reasons/api.py:3859` and `reasons/api.py:3904`.

So, based on the observed lines, `is_derived` is not computed inline at those sites; it is a boolean copied from `has_justifications`. The likely meaning is “this node has justifications,” but the actual computation of `has_justifications` is **not included** in the observations.

The CLI then uses that flag as a gate for trace behavior:

- `reasons/cli.py:2265`:
  ```python
  if result["is_derived"] and getattr(args, "trace", False):
  ```

So trace output is only considered when the API says the result is derived and the user passed `trace=True`.

### Recursive `sl` premise collection and de-duplication

The observations **do not contain the implementation** that recursively collects and de-duplicates `sl` premises for `trace=True`.

Evidence for this:

- `locate_trace_in_derive` found **0 matches** for `trace` in the searched derive-related matches.
- `locate_sl_in_derive` found **0 matches** for `\bsl\b` in the searched derive-related matches.
- The read portion of `reasons/derive.py` is 500 of 697 lines and contains no `trace` logic and no exact lowercase `sl` collection logic. It does mention “SL (support-list)” only in the static prompt text, not in executable trace-collection code.
- The only observed `trace` usage is in the CLI at `reasons/cli.py:2265`, after the API result has already been produced.

The closest recursive graph traversal in the observed `reasons/derive.py` content is `_get_depth`:

```python
def _get_depth(node_id, nodes, derived, memo=None):
    ...
    for j in justifications:
        for a in j.get("antecedents", []):
            max_d = max(max_d, _get_depth(a, nodes, derived, memo))
```

That function recursively walks antecedents and uses a `memo` dict to guard cycles, but it computes **depth**, not a de-duplicated `sl` premise list for trace output.

Similarly, `_build_derived_section` only prints the first justification’s antecedents and outlist:

```python
justs = derived[k]["justifications"]
antes = justs[0].get("antecedents", []) if justs else []
outlist = justs[0].get("outlist", []) if justs else []
```

It does not recursively expand `sl` premises or de-duplicate them.

### Conclusion

From the observations alone:

- `is_derived` is sourced from `has_justifications` in the API response builders at `reasons/api.py:3859` and `reasons/api.py:3904`.
- The CLI gates trace behavior on `result["is_derived"]` plus `args.trace` at `reasons/cli.py:2265`.
- The recursive collection and de-duplication of `sl` premises for `trace=True` is **not visible** in the provided code. To explain it accurately, we would need the omitted portions of `reasons/api.py` around lines 3800–3910, the CLI trace handler after `reasons/cli.py:2265`, and any helper functions that build or serialize trace data.

## Topics to Explore

- [file] `reasons/api.py` — Inspect lines around 3800–3910 to find how `has_justifications` is computed and how trace payloads are assembled.
- [file] `reasons/cli.py` — Read the trace handling immediately after line 2265 to see how the API trace result is rendered.
- [function] `reasons/derive.py:_get_depth` — Shows the observed recursive antecedent traversal and memoization pattern, though it computes depth rather than `sl` trace premises.
- [general] `support-list-premise-trace` — Locate the missing helper that recursively collects and de-duplicates SL/support-list premises for `trace=True`.
- [file] `tests/test_verify.py` — The assertion at line 242 exercises `is_derived` and may reveal the expected API contract.

## Beliefs

- `is-derived-is-has-justifications` — In both observed API response construction sites, `is_derived` is assigned from `has_justifications` (`reasons/api.py:3859`, `reasons/api.py:3904`).
- `cli-checks-is-derived-before-trace` — The CLI only enters trace handling when `result["is_derived"]` is true and `args.trace` is set (`reasons/cli.py:2265`).
- `derive-module-observation-lacks-trace-sl-collection` — The observed portion of `reasons/derive.py` contains no `trace` logic and no exact `sl` matches; SL appears only in prompt text.
- `get-depth-recurses-antecedents` — `_get_depth` recursively traverses justification antecedents and uses a memo dict to guard cycles (`reasons/derive.py:_get_depth`).
- `observed-code-does-not-show-sl-premise-dedup` — The provided observations do not include the implementation that recursively collects and de-duplicates SL premises for `trace=True`.
