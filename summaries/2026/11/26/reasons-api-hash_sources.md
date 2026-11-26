---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: hash_sources in reasons/api.py

# `hash_sources` — Explanation

Defined in `reasons/api.py`.

```python
def hash_sources(
    force: bool = False,
    repos: dict[str, str] | None = None,
    db_path: str = DEFAULT_DB,
    pg_conninfo=None, domain_id=None,
) -> dict:
    """Backfill source hashes for nodes with source paths but no stored hash.

    Returns: {"hashed": list[dict], "count": int}
    """
    if pg_conninfo:
        return _pg_dispatch(pg_conninfo, domain_id, "hash_sources",
                            force=force, repos=repos)
    from pathlib import Path as P
    from .check_stale import hash_sources as _hash

    db_dir = P(db_path).resolve().parent

    with _with_network(db_path, write=True) as net:
        repo_paths = repos
        if repo_paths is None and net.repos:
            repo_paths = net.repos
        if repo_paths:
            repo_paths = {k: P(v) for k, v in repo_paths.items()}

        results = _hash(net, repo_paths, force=force, db_dir=db_dir)
        return {"hashed": results, "count": len(results)}
```

---

## 1. Purpose

`hash_sources` is a thin **public API wrapper** that computes and persists content hashes for the source files referenced by beliefs in the network. It exists so that CLI commands, LangGraph tools, or plain Python callers can "backfill" hashes without managing `Storage` lifecycle, without resolving repo paths themselves, and without knowing whether the backend is SQLite or PostgreSQL.

It is the counterpart to `check_stale` — `check_stale` compares stored hashes against current file content, whereas `hash_sources` *writes* those hashes (typically the first time a belief is imported without one).

## 2. Contract

**Preconditions**
- A reachable database exists at `db_path` (SQLite). If `pg_conninfo` is supplied, the SQLite path and its `db_path` are effectively ignored and the call is forwarded.
- The network's nodes reference sources via `source` strings that the underlying `.check_stale.hash_sources` (and its `resolve_source_path` helper) can resolve against either the passed `repos` map or `net.repos`.
- The process can read the referenced source files from disk (they must exist to be hashable).

**Postconditions**
- If no exception is raised, the network (including any new `source_hash` fields and `node_sources` rows written by the underlying function) has been persisted to the SQLite DB.
- If `pubsub` has subscribers for `db_path`, change events are computed and published on context exit (for truth-value changes only, which for a pure hashing operation are typically empty).
- The `_network_cache[db_path]` entry is refreshed with the new file mtime.

**Invariants**
- The network is loaded fresh because `write=True`, so the caller never operates on a stale cached `Network` for write purposes.
- If an exception is raised inside the `with` block, the network is *not* saved (`__exit__` skips `store.save`), and the cache is *not* updated (`not (write and exc_type is not None)`).

## 3. Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `force` | `bool` | Forwarded to `.check_stale.hash_sources`. When `True`, presumably recomputes hashes even if one is already stored; when `False` (default) it only backfills missing hashes. | Semantics depend entirely on the inner function; API doesn't re-validate. |
| `repos` | `dict[str, str] \| None` | Overrides/extends the repo map used to resolve relative source paths. If `None`, falls back to `net.repos`. If `{}` (empty dict), it is *not* replaced by `net.repos` (`net.repos` is skipped because `repo_paths is None` is false), and the falsy `{}` short-circuits the `Path` coercion — so `{}` is passed as-is to `_hash`, which may treat it differently from `None`. | Caller should pass `None` (or omit) to use the stored repo map, not `{}`. |
| `db_path` | `str` | Path to the SQLite DB. Used both for `Storage(db_path)` and for computing `db_dir = Path(db_path).resolve().parent`, which helps the inner function resolve source paths relative to the DB file. | If the file doesn't exist, `Storage(db_path)` will likely create/init or raise — behaviour determined by `Storage`. |
| `pg_conninfo` | `Any` (conninfo string / object) | If truthy, short-circuits the entire SQLite path and calls `PgApi.hash_sources(force=..., repos=...)`. | Note: `db_path`, `domain_id` are only meaningful in the PG branch for `domain_id`; `db_path` is ignored. |
| `domain_id` | `Any` | Forwarded only to `_pg_dispatch` (and thus `PgApi`). Ignored on the SQLite path. | — |

## 4. Return Value

Two possible shapes:

- **SQLite path**: `{"hashed": list[dict], "count": int}`. `hashed` is whatever the inner `.check_stale.hash_sources` returns — typically a list of `{"node_id", "source_hash", ...}` dicts; `count` is `len(hashed)`.
- **PostgreSQL path**: the return value of `PgApi.hash_sources(...)`. The docstring does not constrain it further; the API trusts the backend.

Callers should treat `hashed` as an opaque, JSON-serializable list (the module docstring says "All functions return dicts suitable for JSON serialization").

## 5. Algorithm

1. **Backend dispatch.** If `pg_conninfo` is set, forward to `PgApi` via `_pg_dispatch` and return immediately.
2. **Late imports.** Import `Path` and the underlying `_hash` (deferred to avoid module-weight at import time and to localize the `check_stale` dependency).
3. **Compute `db_dir`.** `Path(db_path).resolve().parent` — the directory the caller's DB lives in, used as a fallback search root when resolving relative source paths.
4. **Open network for writing.** Enter `_with_network(db_path, write=True)`, which:
   - Loads a fresh `Network` from `Storage(db_path)`.
   - Snapshots `truth_value` per node (only if pubsub has subscribers) so `_with_network` can emit change events on exit.
5. **Resolve repo paths.**
   - Start with `repo_paths = repos` (may be `None`).
   - If `None` and `net.repos` is truthy, fall back to `net.repos`.
   - If still truthy (non-`None`, non-empty), rebuild as `{name: Path(value)}` — this keeps `net.repos` unmodified.
6. **Delegate to `_hash`.** Call `check_stale.hash_sources(net, repo_paths, force=force, db_dir=db_dir)` — this is where the actual iteration over nodes, file reads, `sha256` computation, and assignment of `node.source_hash` / `node_sources` occur.
7. **Return** `{"hashed": results, "count": len(results)}`.
8. **Context exit.** Because the block completed without exception, `_with_network.__exit__` calls `store.save(self.network)`, publishes any truth-value changes to subscribers, and refreshes `_network_cache[db_path]`.

## 6. Side Effects

- **Disk writes (DB)**: The `write=True` context persists the mutated `Network`, including any `source_hash` fields set by the inner function, when it exits cleanly.
- **Disk reads**: The inner `_hash` reads source files from the filesystem. If a repo path is a git checkout, it may invoke git (depending on `hash_sources`'s internals).
- **Pubsub**: On successful exit, `pubsub.compute_changes` / `pubsub.publish` fire if there are subscribers for `db_path`, even though for a pure hashing pass the truth-value delta is normally empty.
- **Cache**: `_network_cache[db_path]` is replaced with `(new_mtime, self.network)`.
- **Module import side effects**: imports of `.check_stale` are deferred to the call site, which keeps `reasons.api` import-light.

## 7. Error Handling

- **Backend reachable?** If `pg_conninfo` is set but `PgApi` import/connect fails, the exception propagates. If `db_path` doesn't exist, `Storage(db_path)` may raise (typically `sqlite3.OperationalError` or a custom `Storage` error) — the exception propagates.
- **Source file missing**: Handled inside `.check_stale.hash_sources`; the API layer does not catch. Typically that function records the miss in its returned list (as `check_stale` does with `"error"`-tagged rows), so the caller gets a partial `hashed` list plus any recorded errors, without an exception being raised.
- **Exception inside the with block**: `_with_network.__exit__` skips `store.save(...)` and skips the cache refresh (`if not (write and exc_type is not None)`), so the DB is not corrupted and the cache is not poisoned with a half-mutated `Network`. The exception continues to propagate.
- **No silent swallowing**: unlike some callers in this module (e.g., `classify_defeat_reason_types` swallows LLM exceptions), `hash_sources` does not catch anything.

## 8. Usage Patterns

- **CLI backfill**: after importing beliefs.md / network.json, run `reasons hash-sources` to fill in `source_hash` for every node with a `source` field. The default `force=False` preserves already-stored hashes.
- **Force recompute**: after a `git reset` in repos, `force=True` recomputes all hashes when the stored ones are suspect.
- **Repo override**: pass `repos={"repoA": "/abs/path/to/repoA"}` when the DB was created on a different machine or the repos moved, overriding `net.repos`.
- **PG backend**: pass `pg_conninfo` to route the same operation through the Postgres implementation.
- **Caller obligations**:
  - Do not assume the operation is atomic across multiple concurrent writers — SQLite + `_with_network` gives last-writer-wins semantics.
  - Do not pass `repos={}` expecting stored map fallback; use `None` for that.
  - Expect the return value's `hashed` entries to be backend-specific; safest to treat as JSON blobs.

## 9. Dependencies

- **Internal**:
  - `_with_network` (same module) — write-mode context manager.
  - `.check_stale.hash_sources` — the actual hashing algorithm.
  - `.check_stale.resolve_source_path` (indirectly, via the hash function) — repo/relative path resolution.
  - `.storage.Storage` (via `_with_network`) — disk persistence.
  - `.pubsub` (via `_with_network`) — change notifications.
  - `.pg.PgApi` (via `_pg_dispatch`) — PostgreSQL backend.
- **External**:
  - `pathlib.Path` — path normalization.
  - Standard library only; no third-party dependencies added at this layer.

---

## Topics to Explore

- [function] `reasons/check_stale.py:hash_sources` — The actual hashing loop; this is where `force`, `db_dir`, and source-file resolution semantics live. Understanding it is required to reason about what `hash_sources` really does.
- [function] `reasons/api.py:_with_network` — The load/save/cache/pubsub context manager that every write-mode API function relies on; explains the atomicity, cache, and change-event behavior of `hash_sources`.
- [file] `reasons/check_stale.py` — Contains `resolve_source_path`, `pin_sources`, and the sibling `check_stale` function; all use the same repo-path-resolution contract.
- [file] `reasons/pg.py` — The `PgApi` class to compare the PostgreSQL implementation of `hash_sources` against the SQLite one; useful for confirming the two backends stay in sync on this API surface.
- [general] `source-hash-lifecycle` — How `source_hash` is written, upgraded (`check_stale(upgrade_hashes=True)`), and checked (integrity/Merkle verification), so you understand where `hash_sources` sits in the lifecycle.

---

## Beliefs

- `hash-sources-requires-write-context` — `hash_sources` always opens the network with `write=True`, so any hashes set by the inner function are persisted to disk on successful context exit (and skipped if an exception interrupts the block).
- `hash-sources-delegates-to-check-stale` — The actual file-reading and SHA computation is performed by `reasons.check_stale.hash_sources`; the API layer only resolves repo paths and manages persistence.
- `hash-sources-falls-back-to-net-repos` — When `repos` is `None` and `net.repos` is non-empty, the function uses `net.repos` (the network's stored repo map) as the resolution source, after converting the values to `Path`.
- `hash-sources-dispatches-to-postgres` — Providing a truthy `pg_conninfo` bypasses the SQLite path entirely and returns the `PgApi.hash_sources` result, ignoring `db_path`.
- `hash-sources-updates-network-cache` — Because it uses `_with_network(write=True)`, a clean run refreshes `_network_cache[db_path]` with the new mtime; a failed run leaves the previous cache entry intact.
