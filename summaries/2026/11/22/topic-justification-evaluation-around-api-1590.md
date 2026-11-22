---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Inspect the exact evaluator using `outlist_ok = all(...)` to see how missing, cyclic, or in-progress nodes are read.

## How `outlist_ok = all(...)` reads the truth cache

The evaluator that decides whether a justification fires is gated by two booleans: `inlist_ok` (antecedents established) and `outlist_ok` (no `unless` node is established). Both must be `True`.

Only **two** sites in the observations show the full `all(...)` expression; the rest only show the opening `outlist_ok = all(` line:

| File | Line | Full expression visible? | Notes |
|---|---|---|---|
| `reasons/pg.py` | 3158 | ✅ | `outlist_ok = all(truth_cache.get(o, "OUT") == "OUT" for o in outs)` |
| `reasons/pg.py` | 3537 | ✅ | `outlist_ok = all(truth_cache.get(o, "OUT") == "OUT" for o in outlist)` |
| `reasons/pg.py` | 3097 | ❌ (only `outlist_ok = all(`) | Followed by `if inlist_ok and outlist_ok:` at 3100 |
| `reasons/api.py` | 1590 | ❌ (only `outlist_ok = all(`) | Followed by `if inlist_ok and outlist_ok:` at 1594 |
| `reasons/network.py` | 987 | ❌ (only `outlist_ok = all(`) | `return inlist_ok and outlist_ok` at 991 |

### What the visible pattern guarantees

At `reasons/pg.py:3158` and `reasons/pg.py:3537` the loop body is:

```python
truth_cache.get(o, "OUT") == "OUT"
```

For every node `o` in the `unless` list:

- **`truth_cache[o] == "OUT"`** → not established → the `unless` clause is satisfied for `o` → `True`.
- **`truth_cache[o] == "IN"`** (or any other non-`"OUT"` sentinel) → the clause is not satisfied → `False`.
- **`o` not present in `truth_cache`** → `dict.get` returns the default `"OUT"` → treated as not established → `True`.

Because the check is `all(...)`, a single `"IN"` (or sentinel) node flips `outlist_ok` to `False`, and the justification is blocked by `inlist_ok and outlist_ok` at `reasons/pg.py:3159`, `reasons/pg.py:3538`, `reasons/pg.py:3100`, `reasons/api.py:1594`, `reasons/network.py:991`.

### Missing nodes — fail-open

The `"OUT"` default means a missing outlist node **does not block** the justification. Absence of evidence is interpreted as evidence of absence in the `unless` list. This is the fail-open branch, and it is testable directly from the two full expressions in `reasons/pg.py`.

### Cyclic / in-progress nodes — not directly observable here

None of the visible `outlist_ok` lines reference in-progress or cyclic state. The only related evidence in the observations is the memoization pattern:

- `reasons/api.py:3378` — `memo[nid] = 0  # cycle guard`
- `reasons/derive.py:123` — `memo[node_id] = 0  # cycle guard`

These show cycles are recorded as `0` in a **derivation memo**, not necessarily in `truth_cache`. Two interpretations remain possible and the observations cannot distinguish them:

1. **Fail-closed for cycles:** if the evaluator writes a non-`"OUT"` sentinel (e.g., `0`, `"IN_PROGRESS"`, `"UNKNOWN"`) into `truth_cache` before recursing, then `truth_cache.get(o, "OUT") == "OUT"` is `False`, and the justification is blocked. That matches the intent of a cycle guard.
2. **Fail-open for cycles:** if `truth_cache` is only written *after* a node resolves, a node currently under evaluation is absent, so the `"OUT"` default fires and `outlist_ok` stays `True` — accidentally treating in-progress work as not-established.

The presence of cycle guards (`memo[...] = 0`) strongly suggests interpretation 1 is the intended behavior, but that's inference, not observation.

### What's missing to answer the cyclic/in-progress question definitively

1. The continuation of `outlist_ok = all(` at `reasons/api.py:1590`, `reasons/network.py:987`, and `reasons/pg.py:3097`. They may use `truth_cache.get(o, "OUT")` or something different.
2. The code that **populates** `truth_cache` — specifically, what value is written for a node currently being evaluated.
3. The values of `inlist_ok` at the equivalent lines (its default and sentinel handling are not shown).
4. The literal sentinel strings used for in-progress work (search for `"IN_PROGRESS"`, `"UNKNOWN"`, `"PENDING"`, or `0` writes into `truth_cache`).

`find_evaluator_definitions` returned **zero** matches, so the evaluator function that builds `truth_cache` itself is not in the observations. The reader must open `reasons/pg.py` around 3097–3540 and search for `truth_cache =` assignments (not just reads) to resolve the cyclic/in-progress semantics.

## Topics to Explore

- [file] `reasons/pg.py` — Contains all three full/partial `outlist_ok` expressions (3097, 3158, 3537); trace `truth_cache` construction around the 3097 evaluator to see if in-progress sentinels are written
- [file] `reasons/api.py` — Parallel in-memory evaluator at 1590 with the same `inlist_ok and outlist_ok` gate; verify whether it also uses `truth_cache.get(o, "OUT")`
- [file] `reasons/network.py` — Network-level evaluator at 987 (`return inlist_ok and outlist_ok` at 991); compare the `all(...)` default to pg.py
- [function] `reasons/derive.py:memoize_cycle` — `memo[node_id] = 0` at line 123 shows the cycle-guard memoization pattern that likely informs `truth_cache`
- [general] `truth-cache-sentinels` — Search the repo for writes into `truth_cache` and for sentinel strings (`IN_PROGRESS`, `UNKNOWN`, `PENDING`) to pin down whether cycles block or allow justifications

## Beliefs

- `missing-outlist-defaults-to-out` — `truth_cache.get(o, "OUT") == "OUT"` at `reasons/pg.py:3158` and `reasons/pg.py:3537` treats any outlist node absent from the cache as `"OUT"`, so a missing node does not block the justification.
- `outlist-ok-is-all-out` — `outlist_ok` is `True` only when every outlist node maps to `"OUT"` in `truth_cache`; any non-`"OUT"` value (including `"IN"` or a sentinel) makes it `False`.
- `justification-fire-requires-inlist-ok-and-outlist-ok` — The gate `inlist_ok and outlist_ok` appears at `reasons/api.py:1594`, `reasons/network.py:991`, `reasons/pg.py:3100`, `reasons/pg.py:3159`, and `reasons/pg.py:3538`.
- `cycle-guard-stores-zero-in-memo` — Both `reasons/api.py:3378` and `reasons/derive.py:123` set `memo[x] = 0` as the cycle guard, showing cycles are memoized to `0` rather than left absent.
- `truth-cache-holds-in-and-out` — The comparison `truth_cache.get(o, "OUT") == "OUT"` in `reasons/pg.py` implies `truth_cache` stores at least `"IN"` and `"OUT"` as terminal truth values, but the in-progress sentinel (if any) is not visible in the observations.
