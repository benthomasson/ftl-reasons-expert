---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Review Doyle 1979 SL/CP semantics to sanity-check `_justification_valid` and whether CP should differ from SL beyond the current no-op.

## Short answer

The codebase documents a real Doyle-style distinction between SL and CP in `reasons/__init__.py:11-13`:

- **SL (Support List)**: node is IN iff all antecedents are IN.
- **CP (Conditional Proof)**: node is IN iff assumptions are consistent.

That is not a label-only distinction in Doyle 1979. If `_justification_valid` only checks antecedents and outlist, then CP is currently a no-op relative to SL. The strongest evidence in the observations is the PostgreSQL validator signature:

```python
# reasons/pg.py:3524
def _justification_valid_cached(self, cur, antecedents, outlist):
```

It receives no `Justification` object and no `type` parameter, and it is called at `reasons/pg.py:3475` as:

```python
if self._justification_valid_cached(cur, ants, outs):
```

So unless that function performs an extra type lookup, it cannot distinguish SL from CP. The network path is less conclusive from the observations alone: `_justification_valid(self, j: Justification)` at `reasons/network.py:976` receives the full justification, so it could inspect `j.type`, but the observed call sites at `reasons/network.py:856`, `860`, and `970` do not reveal the body.

## What the observations show

The `Justification` type is definitely persisted and used during creation:

- `reasons/api.py:326-334` appends `Justification(type="SL", ...)` or `Justification(type="CP", ...)`.
- `reasons/api.py:331` creates CP with the same `antecedents` and `outlist` fields as SL:
  ```python
  justifications.append(Justification(type="CP", antecedents=antecedents, outlist=outlist, label=label))
  ```
- `reasons/api.py:441-460` selects `jtype` and applies `any_mode` only for SL.
- `reasons/api.py:499` uses `j.type == "SL"` for some multi-antecedent check, but that is not the truth-value validator.

There is no observed code that implements “assumptions are consistent” for CP. The `reasons/verify.py` files in the observations are unrelated: they verify beliefs against source documents using an LLM, not TMS justification validity.

## Doyle 1979 sanity-check

Doyle’s TMS justifications are built around an in-list and an out-list. The ordinary SL justification is valid when:

- every in-list node is IN, and
- every out-list node is OUT.

That is the standard non-monotonic support-list check. The codebase’s `outlist` handling is therefore already SL-shaped.

A Doyle CP justification is intended to capture conditional proof under assumptions. Its validity should depend on the consistency of the assumption set, not merely on each antecedent being IN. The codebase’s own docstring states this at `reasons/__init__.py:13`:

```text
- CP (Conditional Proof): node is IN iff assumptions are consistent
```

So CP should differ from SL. It cannot be implemented by reusing the same `(antecedents, outlist)` boolean check, because that check already describes SL. CP needs an assumption set and a consistency predicate—likely tied to the nogood/contradiction machinery referenced in `.claude/skills/reasons/SKILL.md:99`.

## Practical conclusion

If you intend Doyle-style CP semantics, then yes: CP should differ from SL beyond the current no-op. The current implementation appears to store `type="CP"` but validate it exactly like SL, at least on the PostgreSQL path. That is either a bug or an undocumented alias.

If CP is intentionally out of scope, then the docstring at `reasons/__init__.py:13` is misleading and CP should be documented as an alias for SL or removed from the public API. If CP is in scope, then both validators need to be updated, and the PostgreSQL cached validator likely needs a different signature or cache key because CP consistency may depend on global state beyond `(antecedents, outlist)`.

## What is missing

The observations do not include the body of `_justification_valid` at `reasons/network.py:976` or `_justification_valid_cached` at `reasons/pg.py:3524`. The `locate_justification_valid` result also returned no definition even though the usage list shows a `def` line, so the body was not captured. To fully confirm the no-op, inspect those two function bodies and check whether either one branches on `Justification.type` or performs a nogood/consistency check.

## Topics to Explore

- [function] `reasons/network.py:_justification_valid` — Inspect the body to see whether it branches on `j.type` and how it handles `antecedents` versus `outlist`.
- [function] `reasons/pg.py:_justification_valid_cached` — Check whether it can see justification type and whether its cache key is sufficient for CP consistency semantics.
- [file] `reasons/api.py` — Review justification creation and type-specific logic around lines 300-334, 421-460, and 499.
- [general] `nogood-consistency-check` — Determine whether the existing nogood/contradiction machinery can supply the CP “assumptions are consistent” predicate.
- [file] `reasons/__init__.py` — Compare the `Justification` dataclass and its SL/CP docstring against Doyle 1979’s actual justification forms.

## Beliefs

- `pg-cached-validator-lacks-justification-type` — `_justification_valid_cached` in `reasons/pg.py:3524` takes only `cur`, `antecedents`, and `outlist`, so it cannot directly distinguish SL from CP.
- `cp-justifications-store-type-cp` — `reasons/api.py:331` creates `Justification(type="CP", antecedents=antecedents, outlist=outlist, label=label)`.
- `sl-cp-docstrings-differ` — `reasons/__init__.py:11-13` defines SL as all antecedents IN and CP as assumptions consistent.
- `network-validator-receives-full-justification` — `_justification_valid(self, j: Justification)` at `reasons/network.py:976` is called with a `Justification` at lines 856, 860, and 970, so it has access to `j.type`.
- `any-mode-only-expands-sl` — `reasons/api.py:457` only expands multi-antecedent SL justifications when `any_mode` is true; CP justifications are not expanded.
