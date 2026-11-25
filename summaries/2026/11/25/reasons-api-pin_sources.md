---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: pin_sources in reasons/api.py

# `pin_sources` — `reasons/api.py`

## Purpose

`pin_sources` is a thin, mutating façade over `reasons.check_stale.pin_sources`. It pins every **IN** belief that has a `source` (git-tracked file) to the current commit SHA of that file, recording the resulting `pinned_sha` and a `verified_at` timestamp on the node. It exists so CLI tools and scripts can freeze beliefs against the exact source revision they were derived from, enabling future staleness checks to be cheap and precise (`check_stale(git_aware=True)` short-circuits when no commit has touched a pinned file since `pinned_sha`).

It is the API-level counterpart of the same operation that `hash_sources` performs for content hashes — pinning is a *revision* snapshot, hashing is a *content* snapshot.

## Contract

**Preconditions**
- A SQLite database exists at `db_path` (or is at least constructible by `Storage`). Loading a nonexistent DB is delegated to `Storage`/`Network` and is not defended against here.
- The `Network.repos` mapping (or the caller-supplied `repos`) contains entries whose values are filesystem paths to git working trees. Path validity is **not** checked at the API layer.
- The caller accepts that the call rewrites the on-disk database (or at least re-serializes it via `Storage.save` when `write=True`).

**Postconditions**
- For each successfully pinned node, `node.metadata["pinned_sha"]` and `node.metadata["verified_at"]` are set.
- The network is saved back to `db_path` (write=True triggers `Storage.save` on clean exit).
- The in-memory `_network_cache[db_path]` entry is refreshed with the new file mtime.
- Truth values are **not** expected to change — pinning is a metadata operation only, but the write context still runs change-detection/publish if there are pubsub subscribers.

**Invariants**
- Same DB path ⇒ same storage contract as all other mutating API functions (single write, publish-on-change, cache refresh).
- No `pg_conninfo`/`domain_id` parameters exist: this function is **SQLite-only**. Unlike virtually everything else in `api.py`, it cannot be dispatched to PostgreSQL because the pinning algorithm depends on the local git working tree, not on network state.

## Parameters

| Name | Type | Meaning / Edge Cases |
|---|---|---|
| `force` | `bool` (default `False`) | If `True`, re-pin nodes that already have a `pinned_sha` (likely so `_pin` doesn't skip already-pinned nodes). Precise skip semantics live in `check_stale.pin_sources`. |
| `pin_urls` | `bool` (default `False`) | Instructs `_pin` to also pin URL-based sources (e.g., HTTP refs) — behavior defined downstream. |
| `repos` | `dict[str,str] \| None` | Repo name → path map. If `None`, falls back to `net.repos` (persisted in the DB). Values are converted to `Path` before being handed to `_pin`. Empty dict is passed through as-is (falsy check means “no override”). |
| `db_path` | `str` (default `DEFAULT_DB`, `"reasons.db"`) | Path to the SQLite store. Only its parent directory is used to compute `db_dir`, which `_pin` uses to resolve relative `source` paths. |

Assumption not enforced by the type system: the values in `repos` point at real git working copies. If they do not, `_pin` will fail on those entries (or produce error results — the API caller must inspect the returned list).

## Return Value

```python
{"pinned": results, "count": len(results)}
```

- `results` is **exactly** whatever `check_stale.pin_sources` returns: a list of per-node result dicts. The API wrapper performs no filtering.
- `count` is `len(results)`, **not** a count of *successful* pins. If `_pin` includes entries flagging failures (the sibling `pin_update` API explicitly does so via `"error" in r`), those entries are still included in `pinned` and in `count`. Contrast with `pin_update` which subtracts errors from `count`. Callers that care about success must inspect the individual dicts — the wrapper does not disambiguate.

## Algorithm (step-by-step)

1. **Local imports (deferred).** Import `Path` and the real `_pin = check_stale.pin_sources`. The deferral avoids import cycles and keeps the heavy git machinery out of the module import path.
2. **Compute `db_dir = Path(db_path).resolve().parent`.** This is the directory `_pin` will use to resolve relative `source` references against. Note: it is the parent of the database file, not necessarily the project root.
3. **Enter write context** `_with_network(db_path, write=True)`.
   - Because `write=True`, the read cache is bypassed: `Storage(db_path).load()` is called fresh.
   - Pubsub `_before` snapshot is captured if subscribers exist.
4. **Resolve repositories:**
   - `repo_paths = repos` if provided.
   - Else `repo_paths = net.repos` (stored in DB).
   - If non-empty, coerce values to `Path` via a dict comprehension.
   - Otherwise leave `repo_paths` as `None`/falsy so `_pin` handles the no-repos case.
5. **Call `_pin(net, repo_paths, db_dir=db_dir, force=force, pin_urls=pin_urls)`.** `_pin` mutates `net` in place (setting `pinned_sha`/`verified_at` on eligible IN nodes) and returns the per-node results list.
6. **Return** `{"pinned": results, "count": len(results)}`.
7. **Context exit (implicit):**
   - If no exception: `store.save(net)`, compute changes from `_before`, publish events.
   - `store.close()` always runs.
   - On non-error exits the `_network_cache` is updated with the new mtime.

## Side Effects

- **Filesystem writes**: `Storage.save` writes the whole network to `db_path`.
- **Database reads**: fresh load via `Storage` on entry.
- **Cache mutation**: `_network_cache[db_path]` is refreshed on clean exit.
- **Pubsub**: `pubsub.publish(events, db_path)` fires if (a) subscribers are registered and (b) truth values changed. Pinning should not change truth values, so this is a no-op in practice — but the plumbing runs.
- **External git calls**: `_pin` is expected to shell out to `git` to query HEAD / diff information. (Not visible at this layer.)
- **Metadata mutation**: `pinned_sha` and `verified_at` on each node written by `_pin`.

## Error Handling

- **No `try/except`** in the wrapper. Any exception from `_pin`, `Storage.load`, `Storage.save`, or `pubsub` propagates unchanged.
- **`Storage.load` failure** on a missing DB raises whatever `Storage` raises (typically `sqlite3.OperationalError: no such table`), not a friendly `KeyError`.
- **No per-node error translation.** If `_pin` returns dicts with an `"error"` key (as the sibling `pin_update` shape suggests), this function does **not** surface them separately. The caller must inspect `result["pinned"]`.
- **Partial success is possible.** If some repos resolve and others don't, `_pin` may still return a list mixing pinned entries and error entries; the API layer treats them uniformly.
- **Write propagation:** because the context is `write=True`, if any exception escapes during `_pin`, `_with_network.__exit__` skips `save` and the DB is left unchanged.

## Usage Patterns

Typical CLI call:

```python
result = pin_sources(
    force=False,
    pin_urls=False,
    repos={"reasons": "/abs/path/to/repo"},
    db_path="reasons.db",
)
assert result["count"] == len(result["pinned"])
# iter result["pinned"] to detect per-entry errors
```

Caller obligations:

- **Inspect entries**, not just `count`, if correctness matters — the count is list length, not success count.
- **Provide `repos`** if the DB has no persisted repos and the caller's working directory is not the git tree containing the source files (because `db_dir` resolution is DB-relative).
- **Run while the working tree is at the commit you want to pin** — pinning reads the *current* HEAD; it does not check out revisions. Use `pin_update` to bump existing pinned SHAs after a refactor.
- **Expect SQLite only** — do not pass `pg_conninfo`; the signature does not accept it.

## Dependencies

- `pathlib.Path` — path normalization for `db_dir` and repo values.
- `reasons.check_stale.pin_sources` (aliased `_pin`) — the actual pinning algorithm and per-node result schema.
- `_with_network` + `Storage` (from `reasons.storage`) — load/save lifecycle.
- `pubsub` — change detection/publishing as an incidental side effect of the write context.
- Sibling functions `check_stale`, `hash_sources`, `pin_update`, `pin_lines` share the same `_with_network(db_path, write=True)` pattern and the same `db_dir` derivation.

## Topics to Explore

- [file] `reasons/check_stale.py` — Defines `pin_sources`, `pin_update`, `pin_lines`, and `hash_sources`; the actual git-SHA resolution and per-node result schema lives here, which determines exactly what `pinned` entries look like (including whether error dicts are interspersed).
- [function] `reasons/api.py:_with_network` — The read/write lifecycle that governs saving, cache invalidation, and pubsub emission for every mutating API; understanding its mtime-keyed cache is required to reason about fresh-vs-cached loads on subsequent calls.
- [function] `reasons/check_stale.py:check_stale` — The consumer of `pinned_sha`; shows how pinning makes staleness checks cheap when `git_aware=True` and reveals why the pin metadata is stored under `pinned_sha`/`verified_at`.
- [general] `pubsub-events-on-metadata-writes` — Whether the `pubsub.compute_changes` call inside `_with_network` produces events for metadata-only mutations (a pin is expected not to fire events, but the mechanism is worth confirming because it filters by truth_value only).
- [diff] `pin_update vs pin_sources` — Compare how `pin_update` returns `{"updated", "count" (successes only), "errors"}` while `pin_sources` returns `{"pinned", "count" (all results)}`; the inconsistency affects how callers should interpret `count`.

## Beliefs

- `pin-sources-is-sqlite-only` — `pin_sources` has no `pg_conninfo`/`domain_id` parameters and cannot be dispatched to PostgreSQL, unlike most other API functions.
- `pin-sources-preserves-truth-values` — Pinning only mutates node metadata (`pinned_sha`, `verified_at`) and is expected not to alter truth values; the pubsub plumbing runs but should emit no events.
- `pin-sources-loads-fresh-network` — Because it uses `_with_network(db_path, write=True)`, `pin_sources` bypasses the read-only network cache and always reloads from disk before mutating.
- `pin-sources-count-equals-len-results` — The returned `count` is `len(result["pinned"])`, not a count of successful pins; error entries returned by the underlying `_pin` are still included.
- `pin-sources-requires-repos-or-db-state` — If `repos` is `None`, `pin_sources` falls back to `net.repos`; if both are empty, `repo_paths` is left `None` and repo resolution is delegated entirely to `check_stale.pin_sources`.
- `pin-sources-uses-db-parent-as-db-dir` — Relative `source` paths are resolved against `Path(db_path).resolve().parent`, not against the caller's working directory or the repo root.
