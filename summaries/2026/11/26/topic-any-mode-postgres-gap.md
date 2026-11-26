---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Track down where `reasons/pg.py` would need the equivalent per-antecedent expansion to lift the `NotImplementedError` at `api.py:313` and `api.py:432`.

# Where `pg.py` Needs Per-Antecedent Expansion

## Confirming the `NotImplementedError` locations

Both blockers are in `reasons/api.py`:

- **`api.py:313`** — inside `add_node`, in the `if pg_conninfo:` branch:
  ```python
  if pg_conninfo:
      if any_mode:
          raise NotImplementedError("any_mode is not supported with PostgreSQL")
      return _pg_dispatch(pg_conninfo, domain_id, "add_node", ...)
  ```
- **`api.py:432`** — inside `add_justification`, in the same shape:
  ```python
  if pg_conninfo:
      if any_mode:
          raise NotImplementedError("any_mode is not supported with PostgreSQL")
      return _pg_dispatch(pg_conninfo, domain_id, "add_justification", ...)
  ```

These are the only two `any_mode` gates in `api.py` (the SQLite branches on lines ~323–327 and ~430 are the *reference* behavior that must be mirrored).

## What `any_mode` actually does (the reference behavior)

The SQLite branch is the spec. For `add_node`:

```python
if sl:
    antecedents = [a.strip() for a in sl.split(",")]
    if any_mode and len(antecedents) > 1:
        for a in antecedents:
            justifications.append(Justification(type="SL", antecedents=[a], outlist=outlist, label=label))
    else:
        justifications.append(Justification(type="SL", antecedents=antecedents, outlist=outlist, label=label))
elif cp:
    ...
```

Two invariants to preserve:

1. **Only SL is expanded.** CP is always kept as a single justification with all its antecedents (the `elif cp:` branch ignores `any_mode`).
2. **Expansion produces one justification per antecedent**, each carrying the same `outlist` and `label`. This gives OR semantics: any single antecedent that is IN satisfies the node.

`add_justification` (SQLite, lines ~428–437) does the same thing and hard-codes `premise_count = 1` in the `any_mode` return.

## The pg-side dispatch path

`_pg_dispatch(...)` (in `api.py`) instantiates `PgApi` and calls the named method with the kwargs passed through. So lifting line 313 / 432 requires three coordinated changes:

### 1. `PgApi.add_node` (top of class, ~line 200)

Currently:
```python
def add_node(self, node_id, text, sl="", cp="", unless="", label="",
             source="", source_url="", access_tags=None,
             namespace=None, example=None):
    ...
    justifications = self._parse_justifications(sl, cp, unless, label)
```

Needs:
- An `any_mode=False` parameter.
- Pass it through to `_parse_justifications(sl, cp, unless, label, any_mode=any_mode)`.
- Because `_add_node_raw` already inserts one row per dict in `justifications` (see the `for j in justifications:` loop inserting into `rms_justifications`), the storage layer is already OR-capable — **no schema or insert change is required**.

### 2. `PgApi.add_justification` (~line 290)

Same change: add `any_mode=False`, forward to `_parse_justifications`. The existing loop already inserts one row per dict and calls `_compute_truth` only *after* the loop, so multiple rows already behave as OR — again, no insert-time changes are needed.

### 3. `PgApi._parse_justifications` — **this is the actual expansion site**

This method is *not included in the observations*, but two call sites and `_add_node_raw`'s field access (`j["type"]`, `j.get("antecedents", [])`, `j.get("outlist", [])`, `j.get("label", "")`) pin down its contract: it returns a **list of dicts** shaped like `Justification` (`justifications` from the observation).

The missing branch is roughly the pg equivalent of the SQLite SQLite lines 323–327:

```python
def _parse_justifications(self, sl, cp, unless, label, any_mode=False):
    outlist = [o.strip() for o in unless.split(",") if o.strip()] if unless else []
    justifications = []
    if sl:
        antecedents = [a.strip() for a in sl.split(",")]
        if any_mode and len(antecedents) > 1:
            for a in antecedents:
                justifications.append({"type": "SL", "antecedents": [a],
                                       "outlist": outlist, "label": label})
        else:
            justifications.append({"type": "SL", "antecedents": antecedents,
                                   "outlist": outlist, "label": label})
    elif cp:
        antecedents = [a.strip() for a in cp.split(",")]
        justifications.append({"type": "CP", "antecedents": antecedents,
                               "outlist": outlist, "label": label})
    elif outlist:
        justifications.append({"type": "SL", "antecedents": [],
                               "outlist": outlist, "label": label})
    return justifications
```

That's the entire per-antecedent expansion.

### 4. `api.py` line 313 / 432 themselves

Once the pg method accepts `any_mode`, remove the raise and add `any_mode=any_mode` to the `_pg_dispatch(...)` kwargs on **lines 314–318** and **around line 433–437**. Without this, the block is still reachable even after pg is fixed.

## Why nothing else needs to change

- **`_validate_refs`** is already called on the full `justifications` list. After expansion each justification has a single antecedent — every antecedent still gets validated (and this is *strictly harder to sneak past*, since a single nameless antecedent on its own is still surfaced with the same `KeyError` as the SQLite path).
- **`_inherit_access_tags(cur, node_id, justifications)`** already walks the list, so union inheritance across expanded justifications works.
- **`_compute_truth(cur, node_id)`** is called *after* all rows are inserted in both `_add_node_raw` and `add_justification`, and tests like `test_multiple_justifications` (in `tests/test_pg.py`, `TestPropagation`) confirm it implements OR across `rms_justifications` rows already.
- **`rms_justifications` schema** already stores a single justification per row with `antecedents JSONB`, so expanding to more rows is a pure data-shape change, not a schema change.
- **`Search` / `list_*` / `explain_node`** consume rows from `rms_justifications`, so they see the expanded rows transparently.

## What the observations don't cover

The observations are missing the exact bodies of:

- `PgApi._parse_justifications` — the method that must change.
- `PgApi._compute_truth` — to confirm the OR-over-multiple-justifications invariant (the tests imply it, but the code isn't shown).
- `PgApi._validate_refs`, `PgApi._inherit_access_tags` — to confirm they tolerate a longer `justifications` list.

Also note the two grep probes (`find_not_implemented_errors`, `find_antecedent_references`) returned zero matches even though both strings are visibly present in the pasted code — those searches appear to have been scoped or configured incorrectly and should not be relied on.

## Topics to Explore

- [function] `reasons/pg.py:PgApi._parse_justifications` — The actual expansion site; confirm the current shape and add the `any_mode` branch matching SQLite `api.py:add_node` lines 323–327.
- [function] `reasons/pg.py:PgApi._compute_truth` — Verify OR semantics across multiple `rms_justifications` rows, which is what makes per-antecedent expansion work without storage changes.
- [function] `reasons/pg.py:PgApi._add_node_raw` — Confirms that `justifications` is already a per-row insert loop, so the expansion is purely a parsing concern.
- [file] `tests/test_pg.py` — The `TestAddJustification::test_add_justification_changes_truth` and `TestPropagation::test_multiple_justifications` tests are the closest parity checks; new `any_mode` tests should live alongside them.
- [general] `pg-vs-sqlite-add-node-return-shape` — `PgApi.add_node` returns a `premise_count` computed as the count of premise *nodes* in the domain, while SQLite returns `max(len(j.antecedents))`; worth reconciling independent of `any_mode`, since `api.py` callers may rely on it.

## Beliefs

- `api-any-mode-raises-in-pg-path` — Both `reasons/api.py:add_node` (line 313) and `reasons/api.py:add_justification` (line 432) raise `NotImplementedError` whenever `pg_conninfo` and `any_mode=True` are both set.
- `any-mode-expands-sl-only` — `any_mode` in the SQLite path expands only `sl` into one justification per antecedent, never `cp`, and each expanded justification copies the same `outlist` and `label`.
- `pg-parser-is-expansion-site` — The equivalent expansion must be added inside `PgApi._parse_justifications`, since both `PgApi.add_node` and `PgApi.add_justification` derive their `justifications` list exclusively from that method.
- `pg-storage-is-already-or-capable` — `_add_node_raw` and `PgApi.add_justification` already insert one `rms_justifications` row per justification dict and only call `_compute_truth` after the loop, so multiple justifications naturally produce OR semantics without schema changes.
- `api-dispatch-forwarding-required` — Lifting the two raises also requires the `_pg_dispatch(..., "add_node", ...)` and `_pg_dispatch(..., "add_justification", ...)` calls to forward `any_mode`, otherwise the `.pg` side is unreachable from the API layer.
