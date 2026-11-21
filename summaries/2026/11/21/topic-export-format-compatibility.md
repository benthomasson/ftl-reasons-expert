---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Investigate whether `export_network` output between the mirror packages `reasons/` and `reasons_lib/` is identical, since the repo tracks both trees.

## Investigation Summary

Based on the observed code, there is **no direct evidence that `export_network` output is identical** between `reasons/` and `reasons_lib/`, and the current observations are **not sufficient to prove parity**.

Key findings:

- The only observed definition of `export_network` is in `reasons/api.py:1814`.
- That function delegates to the backend at `reasons/api.py:1821`:
  ```python
  return _pg_dispatch(pg_conninfo, domain_id, "export_network", ...)
  ```
- The observed backend implementation is `reasons/pg.py:1342`:
  ```python
  def export_network(self, visible_to=None):
  ```
  So the actual output is produced by the PostgreSQL backend, not directly by `network.py`.
- `reasons_lib/network.py` is only a 3-line compatibility shim:
  ```python
  """Backwards-compatibility shim — use ``from reasons.network`` instead."""
  from reasons.network import *  # noqa: F401,F403
  ```
  This does **not** establish anything about `export_network`, because `export_network` lives in `api.py`, not `network.py`.
- All listed `export_network` matches are under `reasons/`; none are under `reasons_lib/`. However, the match list may be truncated (`total_count: 112`), so this is suggestive rather than conclusive.
- The search for mirror-comparison tests:
  ```text
  pattern: reasons_lib|export_network|mirror
  matches: []
  ```
  found no tests that assert `reasons/` and `reasons_lib/` produce identical `export_network` output.

## Conclusion

The safest conclusion is:

> `reasons_lib/` appears to be a compatibility mirror/shim package, at least for `network.py`. If `reasons_lib/api.py` also re-exports `reasons.api.export_network`, then output is identical by construction for identical inputs and database state. If `reasons_lib/api.py` or `reasons_lib/pg.py` contains an independent or stale implementation, output can diverge.

What is missing:

- Contents of `reasons_lib/api.py`
- Contents of `reasons_lib/pg.py`
- Whether `reasons_lib/__init__.py` re-exports from `reasons`
- Whether any sync/generation mechanism maintains the mirror
- Any test that compares exported network JSON between the two package trees

Also note that even with identical code, exported output depends on runtime state: `db_path`, `pg_conninfo`, `visible_to`, database contents, and backend version. `reasons/api.py:2376` serializes with `indent=2, sort_keys=True`, which normalizes key ordering in JSON output but does not guarantee identical content.

## Topics to Explore

- [file] `reasons_lib/api.py` — Determine whether `export_network` is imported/re-exported from `reasons.api` or independently implemented.
- [file] `reasons_lib/pg.py` — Check whether the PostgreSQL backend implementation delegates to `reasons.pg` or contains a copied/stale `export_network`.
- [function] `reasons/api.py:export_network` — Canonical API entry point and the dispatch path through `_pg_dispatch`.
- [function] `reasons/pg.py:export_network` — Actual backend method that determines exported network shape and content.
- [diff] `reasons vs reasons_lib` — Run a focused tree diff of `api.py`, `pg.py`, and `network.py` to detect drift between mirrored packages.

## Beliefs

- `export-network-canonical-definition-in-reasons-api` — The only observed `export_network` definition is `reasons/api.py:1814`.
- `reasons-lib-network-is-compatibility-shim` — `reasons_lib/network.py` re-exports all symbols from `reasons.network` via `from reasons.network import *`.
- `export-network-delegates-to-pg-backend` — `reasons/api.py:export_network` calls `_pg_dispatch(..., "export_network", ...)` at line 1821.
- `export-network-output-depends-on-database-state` — The exported network comes from the PostgreSQL backend implementation at `reasons/pg.py:1342`.
- `mirror-parity-is-not-covered-by-observed-tests` — The test search for `reasons_lib`, `export_network`, or `mirror` returned zero matches.
