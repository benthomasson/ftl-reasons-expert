---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Investigate whether nogoods are ever re-checked during propagation, or only at `add_nogood` call time.

## Investigation

**Short answer:** Based on the supplied observations, nogoods appear to be checked/applied **at `add_nogood` call time**, not during ordinary propagation. However, the observations are not sufficient to prove this conclusively because the full `reasons/network.py` nogood implementation and the PG backend are not shown.

### Evidence for add-time handling

- `reasons/api.py:1759` defines the public `add_nogood(node_ids, db_path=...)` API.
- `reasons/api.py:1766` dispatches the operation as `"add_nogood"` via `_pg_dispatch(...)`.
- In the in-memory path, `reasons/api.py:1778` calls:
  - `changed = net.add_nogood(node_ids)`
- Immediately after, `reasons/api.py:1779` reads:
  - `ng = net.nogoods[-1]`
- The returned payload documented at `reasons/api.py:1763` includes:
  - `"nogood_id"`
  - `"nodes"`
  - `"changed"`
  - `"backtracked_to"`

This strongly suggests that adding a nogood synchronously applies it, computes affected nodes, and performs any backtracking during the `add_nogood` call itself.

### Evidence about propagation

- `reasons/api.py:773` defines `propagate(db_path=...)`.
- `reasons/api.py:776` dispatches a separate `"propagate"` operation:
  - `return _pg_dispatch(pg_conninfo, domain_id, "propagate")`
- `reasons/cli.py:3277` describes the CLI `propagate` command as:
  - `"Recompute all truth values"`
- `reasons/import_agent.py:317` and `reasons/import_agent.py:489` use `network.recompute_all()` for propagation-related work.
- `reasons/network.py` is described at line 1 as a:
  - `"Dependency network with automatic truth value propagation"`
- Visible propagation methods include:
  - `reasons/network.py:_propagate` invoked at lines `178`, `201`, and `485`
  - `reasons/network.py:_propagate_access_tags` at line `218`

The supplied grep results for `"nogood"` contain no visible `reasons/network.py` matches, and in particular no visible nogood checks inside the propagation methods. That is consistent with propagation being about recomputing truth values rather than re-validating stored nogoods.

### Important limitation

The observations are **not sufficient to conclusively prove that nogoods are never re-checked during propagation**:

- `grep_nogood_terms.total_count` is `592`, but only a subset of matches is shown.
- `find_add_nogood_symbol` only reports `reasons/api.py:1759`, even though `reasons/api.py:1778` calls `net.add_nogood(node_ids)`. The definition of `Network.add_nogood` is therefore not captured in the provided symbol results.
- The PG backend implementation behind `_pg_dispatch(..., "add_nogood", ...)` and `_pg_dispatch(..., "propagate")` is not shown.
- The full body of `reasons/network.py:_propagate` is not shown.
- `find_nogood_tests` returned zero matches, so there is no visible test coverage confirming either behavior.

So the best-supported conclusion is: **the visible code path treats nogoods as an add-time operation; ordinary propagation is described and structured as truth-value recomputation, but the full codebase would need to be inspected to rule out hidden nogood checks.**

## Topics to Explore

- [file] `reasons/network.py` — Inspect `add_nogood`, `_propagate`, and `recompute_all` to see whether propagation ever iterates `self.nogoods` or validates stored nogoods.
- [function] `reasons/network.py:add_nogood` — Confirm exactly when nogood matching, affected-node calculation, and backtracking occur.
- [function] `reasons/network.py:_propagate` — Verify whether the propagation cascade contains any nogood re-checking or only truth-value updates.
- [general] `pg-backend-propagation` — Trace `_pg_dispatch(..., "add_nogood")` and `_pg_dispatch(..., "propagate")` to compare PostgreSQL behavior with the in-memory network.
- [general] `nogood-test-coverage` — Since the observed test search found no `nogood` matches, add tests distinguishing add-time application from propagation-time re-checking.

## Beliefs

- `api-add-nogood-dispatches-to-backend` — `reasons/api.py:add_nogood` delegates either to `_pg_dispatch(..., "add_nogood", ...)` or to `net.add_nogood(node_ids)`.
- `add-nogood-returns-changed-and-backtracked` — `reasons/api.py:add_nogood` returns `changed` and `backtracked_to`, indicating nogood application/backtracking happens during the add call.
- `propagate-is-truth-value-recomputation` — `reasons/api.py:propagate` dispatches a `"propagate"` operation described by the CLI as “Recompute all truth values”.
- `network-propagation-has-no-visible-nogood-reference` — The supplied grep results show no `nogood` references in `reasons/network.py` propagation methods.
- `nogood-test-search-is-empty` — The observed test search for `nogood` returned zero matches.
