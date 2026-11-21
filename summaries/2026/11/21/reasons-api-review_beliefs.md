---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: review_beliefs in reasons/api.py

# `review_beliefs` — Explanation

## Purpose

`review_beliefs` is the Python API entry point for LLM-assisted review of *derived* beliefs in the RMS (Reason Maintenance System). A "derived belief" is a node with one or more justifications — i.e., it was inferred from antecedents rather than asserted as a premise. The function selects a subset of those derived beliefs (filtered by callers' ids, namespace, depth, antecedent dependency, tags, or random sample), sends them to a reviewer backed by an LLM, and — unless `dry_run=True` — stamps `reviewed_at`/`updated_at` timestamps and a `review_result` metadata classification (`invalid`/`insufficient`/`unnecessary`/`pass`) on each successfully reviewed node.

It exists to give CLI commands, LangGraph tools, and scripts a standalone, JSON-serializable way to audit reasoning soundness without managing DB lifecycles themselves, mirroring how the rest of `api.py` wraps storage/scoring concerns.

## Contract

**Preconditions**
- `db_path` points to an existing SQLite database (otherwise `Storage.load()` will fail).
- If `belief_ids` is provided, they need not be derived or exist — non-matching ids are silently dropped, so a caller passing only nonexistent ids ends up with an empty review.
- `model` should be a model identifier recognized by `.llm.invoke_model` (via `.review.review_beliefs`).
- The caller must be prepared to absorb LLM cost/rate limits: the function *actually runs the LLM* even when `dry_run=True`; only the DB writeback is skipped.

**Postconditions**
- Returns a dict with `results`, `reviewed`, `invalid`, `insufficient`, `unnecessary`, `total_derived`, `timestamp`.
- When `dry_run=False` and results are non-empty, each reviewed node that is still present in the DB gets:
  - `reviewed_at = <now>`
  - `updated_at = <now>`
  - `metadata["review_result"] = _classify_review_result(r)`
- Truth values are *not* touched, so no TMS propagation occurs and no pubsub events fire.

**Invariants**
- `total_derived` counts **all** IN derived beliefs across the whole network, independent of filters.
- `reviewed == len(review_ids)` is the count of candidates submitted, **not** the count of items the LLM returned results for. If the LLM drops some ids, `reviewed` will be larger than `len(results)`.
- `invalid + insufficient + unnecessary` are computed from `results` (LLM outputs), not from candidates — so they can undercount when the LLM returns fewer results than were submitted.

## Parameters

| Param | Meaning | Edge cases |
|---|---|---|
| `belief_ids` | Whitelist of ids to consider | Intersected with the derived-belief candidate set. Passing ids of premises or OUT nodes (when `include_out=False`) silently yields nothing. |
| `model` | LLM model name | Passed straight through to `_review` → `invoke_model`. |
| `timeout` | LLM request timeout in seconds | Not validated; caller must choose sane value. |
| `min_depth` | Minimum inference depth (`_get_depth ≥`) | Computed with a shared memo dict; expects `_get_depth` to tolerate OUT nodes when `include_out=True`. |
| `depends_on` | Require *direct* antecedent match | Literal id membership; only inspects each node's justifications, not transitive. No namespace resolution applied. |
| `namespace` | Filter by id prefix | `namespace=""` selects only non-namespaced ids (`":" not in k`); any other value selects `k.startswith(f"{namespace}:")`. |
| `sample` | Random subsample size | Uses Python's global `random` (no seed control) → non-deterministic between calls. Only applies when candidate count exceeds `sample`. |
| `visible_to` | Access-tag filter | Node is kept if it has *no* `access_tags` **or** all its tags are subsets of `visible_to`. Mirrors `_is_visible` semantics. |
| `dry_run` | Skip DB writeback | LLM still runs; `on_batch` still fires. Timestamps in returned dict are still populated. |
| `on_batch` | Progress callback | Opaque `Callable`; passed to `.review.review_beliefs`. Called during review (per batch). |
| `include_out` | Include OUT derived beliefs | Changes candidate universe but `total_derived` remains IN-only. |
| `db_path` | SQLite path | No `pg_conninfo` parameter — this function is SQLite-only (unlike most of `api.py`). |

## Return Value

```python
{
    "results": [...],           # raw result dicts from .review.review_beliefs,
                                # each with at least {"id", "valid", "sufficient", "necessary"}
    "reviewed": int,            # count of candidates submitted (== len(review_ids))
    "invalid": int,             # count where result["valid"] is falsy
    "insufficient": int,        # count where result["sufficient"] is falsy
    "unnecessary": int,         # count where result["necessary"] is falsy
    "total_derived": int,       # IN derived beliefs in the whole network (unfiltered)
    "timestamp": str,           # ISO 8601 UTC "YYYY-MM-DDTHH:MM:SS+00:00"
}
```

Callers must handle:
- Empty `results` (no candidates match filters, or `_review` returned nothing).
- Partial results: `reviewed` ≥ `len(results)` is allowed and normal under LLM failures.
- The overlap of counts is **not** disjoint — a result with `valid=False` and `sufficient=False` counts in both `invalid` and `insufficient`.

## Algorithm

1. **Snapshot the network.** Call `export_network(db_path=db_path)` to get a fully serializable dict of nodes. The returned node dicts are fresh copies, so subsequent mutations won't touch the cached `Network` object.
2. **Compute `all_derived`.** Filter to IN nodes with at least one justification. Save `total_derived` before further filtering.
3. **Build candidate set.**
   - Default: same as `all_derived`.
   - If `include_out`: any node with justifications (IN or OUT).
4. **Filter pipeline** (in this order — reordering changes results):
   - `belief_ids`: exact id whitelist.
   - `visible_to`: tag-subset filter.
   - `min_depth`: `_get_depth(id, nodes, all_derived, memo) >= min_depth`. Shared `memo` across nodes for O(N) work.
   - `depends_on`: direct antecedent containment.
   - `namespace`: prefix (or non-prefix when empty).
   - `sample`: random subsample of the *sorted* key list (sorting makes it deterministic *given* the RNG state, but the RNG is unseeded).
5. **Delegate to the LLM reviewer.** `review_ids = sorted(candidates.keys())`, then `_review(nodes, belief_ids=review_ids, model=..., timeout=..., on_batch=...)`. The full `nodes` dict (not just candidates) is passed — reviewers need context from antecedents.
6. **Writeback (unless `dry_run` or empty results).** Compute `now`, then open `_with_network(write=True)` — this reloads the network *fresh* (writes bypass the cache). For each `nid` that still exists **and** has a corresponding LLM result:
   - `reviewed_at = updated_at = now`
   - `metadata["review_result"] = _classify_review_result(result)`
   The `nid in net.nodes` guard handles the case where a node was deleted between export and writeback. Missing results (LLM skips) are also skipped here.
7. **Summarize** and return.

`_classify_review_result` uses a strict precedence:
```
if not valid       -> "invalid"
elif not sufficient -> "insufficient"
elif not necessary  -> "unnecessary"
else                -> "pass"
```
So a result failing multiple checks is labeled only with the highest-priority failure.

## Side Effects

- **Reads the full network twice** when writes occur: once via `export_network` (cached read path) and once via `_with_network(write=True)` (fresh read). This is deliberate — the writeback happens against a current snapshot, not a stale cache.
- **Calls the LLM** via `.review.review_beliefs` → `invoke_model`, once per batch (`on_batch` fires per batch). Cost is incurred even when `dry_run=True`.
- **Mutates DB rows** on non-dry runs: two timestamps and one metadata field per reviewed node.
- **Triggers `_with_network.__exit__` save + close**, but since truth values are unchanged, `pubsub.compute_changes` yields no events (assuming `pubsub.has_subscribers` triggered the `_before` snapshot).
- **Does not call `net.recompute_all()`** — no propagation is triggered, since no scorable state changed.
- **Populates `_network_cache[db_path]`** on `_with_network` exit, so subsequent reads see the freshly-saved metadata.

## Error Handling

- **`FileNotFoundError` / `sqlite3.OperationalError`** from `Storage(db_path)` if the DB doesn't exist or schema is missing. Propagated.
- **`KeyError`** from `_with_network` internals if the DB is corrupted. Propagated.
- **LLM/network errors** from `_review` (e.g., API failures, timeouts) propagate — the function does *not* try/except around the reviewer. Callers running batch operations must wrap this themselves.
- **`_get_depth` recursion**: the shared `memo` does carry a cycle guard (`memo[nid] = 0` before recursing), so cyclic justifications are handled, but the memo is created *only* when `min_depth` is set — so other filters won't trip that path.
- **Missing result entries** (LLM returned fewer than requested) are silently tolerated — no writeback for that node, no warning.
- **No `pg_conninfo` branch**: unlike almost every other API function, calling this against a Postgres deployment is impossible — there's no dispatch path. This is a real, undocumented limitation.

## Usage Patterns

Typical calls:

```python
# Full audit of every IN derived belief
report = review_beliefs(db_path="reasons.db")

# Dry-run to inspect only
report = review_beliefs(dry_run=True, model="gpt-4")

# Sample 20 beliefs for spot-checking
report = review_beliefs(sample=20, model="claude", timeout=600)

# Review only beliefs downstream of a suspicious premise
report = review_beliefs(depends_on="user-auth-lib-v1", min_depth=2)

# Review everything in a namespace, including currently-OUT derived beliefs
report = review_beliefs(
    namespace="navy", include_out=True, dry_run=False,
    on_batch=lambda i, total, ids: print(f"batch {i}/{total}"),
)
```

Caller obligations:
- Persist the returned `timestamp`/`results` if you want an audit trail — the DB only stores `review_result` and `reviewed_at`, not the raw LLM payload.
- Re-run `review_premises` (or `repair`) to act on flagged results — this function *classifies* but never *modifies* truth values.
- Use `dry_run=True` for a preview; remember the LLM cost is the same.
- Serialize return value is JSON-safe (`datetime` already converted to ISO string).

## Dependencies

- **`.export_network`** (self-module) — produces the tournament snapshot.
- **`.derive._get_depth`** — computes inference depth for the `min_depth` filter.
- **`.review.review_beliefs`** — the actual LLM-batch reviewer.
- **`_with_network`** (self-module) — load/save context manager, with pubsub on truth-value changes.
- **`datetime.now(timezone.utc)`** — timestamp source.
- **`random.sample`** — sub-sampling; state comes from the caller's global RNG.

## Topics to Explore

- [function] `reasons/review.py:review_beliefs` — The actual per-batch LLM reviewer; defines the `valid`/`sufficient`/`necessary` schema this function assumes.
- [function] `reasons/derive.py:_get_depth` — How inference depth is computed and how the shared memo avoids O(N²) work; also reveals whether OUT nodes are safe inputs.
- [file] `reasons/repair.py` — Downstream consumer that takes `review_beliefs` output and either rewrites/retracts flagged beliefs (`repair`, `repair_smuggled`).
- [general] `pg-parity-gaps` — Which `api.py` functions lack a `pg_conninfo` dispatch branch (this one, plus tag/source helpers), and what that implies for Postgres deployments.
- [general] `_review_result-priority` — The four-way `_classify_review_result` precedence and how it affects downstream filtering (e.g., `list_nodes(never_reviewed=True)` ignores `review_result` entirely).

## Beliefs

- `review-beliefs-requires-sqlite-only` — `review_beliefs` has no `pg_conninfo`/`domain_id` parameters and therefore cannot run against the PostgreSQL backend.
- `review-beliefs-invokes-llm-under-dry-run` — `dry_run=True` still executes the LLM reviewer; only the DB writeback is skipped.
- `reviewed-count-is-not-results-length` — The returned `reviewed` field equals `len(review_ids)` (submitted candidates), not `len(results)`, so it can exceed the number of LLM verdicts.
- `total-derived-is-unfiltered` — `total_derived` counts all IN derived beliefs regardless of `belief_ids`, `namespace`, `visible_to`, or other filters.
- `review-result-priority-invalid-first` — `_classify_review_result` maps failures in a fixed precedence: `invalid` > `insufficient` > `unnecessary` > `pass`, so multi-failure results are labeled only by the first.
- `review-beliefs-does-not-propagate` — The writeback only mutates `reviewed_at`, `updated_at`, and `metadata["review_result"]`; no TMS recompute or pubsub events are triggered.
- `review-beliefs-uses-unseeded-random` — The `sample` parameter uses `random.sample` against the caller's global RNG state, making subsampling non-deterministic across runs.
