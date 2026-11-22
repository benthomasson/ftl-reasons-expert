---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: check_stale in reasons/api.py

# `check_stale` — `reasons/api.py`

## Purpose

`check_stale` is the public, functional entry point for detecting whether the **source files that back currently-IN beliefs have changed** since their contents were hashed. It is the "are my beliefs still anchored to their evidence?" check.

The function is deliberately thin: it opens the network with the correct write mode for the requested options, resolves repo paths, computes a count of eligible nodes, delegates the real work to `reasons.check_stale.check_stale`, and reshapes the return into a JSON-friendly summary. This is the same pattern used across `reasons/api.py`: *the `api` module owns lifecycle (open/load/save/close) while a sibling module owns the domain logic.*

## Contract

- **Preconditions**
  - `db_path` points to an existing SQLite database (or `pg_conninfo`/`domain_id` are supplied for the PostgreSQL backend).
  - Any `repos` paths resolve to directories containing git repos or plain files.
  - `repos` and `net.repos` values are strings (they are wrapped in `Path(...)` unguarded).
- **Postconditions**
  - Returns a dict with keys `stale`, `checked`, `stale_count`, `upgraded`, `sha_bumped`.
  - No truth values are modified by this function.
  - If `upgrade_hashes` or `git_aware` was set, the network may have been saved back to disk (hash upgrades require a write).
- **Invariants**
  - `checked == stale_count + <non-stale count>` where `stale_count == len(stale)` — `checked` counts the *eligible* nodes (IN + has `source` + has `source_hash`), while `stale` only contains those that actually diverged.
  - Only nodes with both `truth_value == "IN"`, a non-empty `source`, and a non-empty `source_hash` are eligible for checking.

## Parameters

| Parameter | Meaning | Edge cases |
|---|---|---|
| `repos: dict[str,str] \| None` | Mapping of logical repo name → filesystem path used to resolve each node's `source`. | When `None`, falls back to `net.repos` (the persisted repo map). When both are empty, `repo_paths` stays `None` and the delegate must resolve paths relative to `db_dir`. |
| `upgrade_hashes: bool` | When `True`, a stored hash that is a **prefix** of the current full hash is upgraded in place (legacy truncated hashes). | Implies `write=True` on the network context so upgrades are persisted. |
| `git_aware: bool` | When `True`, nodes with a `pinned_sha` can skip content hashing if `git log` shows no commits touching that file since the pinned SHA. | Also implies `write=True` because `pinned_sha` metadata can be bumped. |
| `db_path: str` | Path to the SQLite database. | Only used to derive `db_dir = Path(db_path).resolve().parent` when resolving relative sources. |
| `pg_conninfo`, `domain_id` | If `pg_conninfo` is truthy, the call is fully dispatched to the PG backend and the SQLite path is skipped entirely. | `domain_id` is required in practice but not enforced by the signature. |

## Return Value

```python
{
    "stale": list[dict],      # per-node staleness findings from the delegate
    "checked": int,           # IN nodes that had both source AND source_hash
    "stale_count": int,       # len(stale)
    "upgraded": int,          # hashes upgraded in place (only if upgrade_hashes)
    "sha_bumped": int,        # pinned_sha bumped (only if git_aware)
}
```

Callers should treat the shape as stable but must be prepared for `stale` items to be backend-defined dicts (the docs in `check_stale.py` describe the fields). `upgraded` and `sha_bumped` will be `0` when the corresponding flags are off.

## Algorithm (step-by-step)

1. **PG shortcut.** If `pg_conninfo` is set, forward to `_pg_dispatch` and return immediately. No local imports of `pathlib` or `check_stale` occur.
2. **Lazy imports.** Pull in `pathlib.Path as P` and the delegate `reasons.check_stale.check_stale as _check`. Both are inside the function to avoid pulling `check_stale` (which shells out to `git`/`hashlib`) at import time.
3. **Resolve `db_dir`.** `Path(db_path).resolve().parent` — used by the delegate for relative source paths in nodes without a repo prefix.
4. **Choose write mode.** `needs_write = upgrade_hashes or git_aware`. Only these two flags can cause persisted mutations.
5. **Open network.** `_with_network(db_path, write=needs_write)` returns a cached `Network` when read-only and the file's mtime hasn't changed; otherwise it loads fresh (and on exit, saves if `write=True`).
6. **Resolve repo paths.** Prefer the caller's `repos`; otherwise use `net.repos`. Then wrap each value in `Path`.
7. **Count eligible nodes.** Iterate `net.nodes.values()` and count those with `truth_value == "IN"` and non-empty `source` and `source_hash`. This is the `checked` number returned to the caller — it reflects eligibility, not the delegate's work list.
8. **Delegate.** Call `_check(net, repo_paths, db_dir=db_dir, upgrade_hashes=..., git_aware=...)`, which returns `(results, upgraded, sha_bumped)`.
9. **Pack and return.** Compose the summary dict; `stale_count` is derived as `len(results)`.

## Side Effects

- **Network cache:** `_with_network` may update `_network_cache[db_path]` on exit with a fresh `(mtime, network)` tuple.
- **Database writes (conditional):** Only when `upgrade_hashes` or `git_aware` is `True` does the context manager `write=True`, causing `Storage.save(self.network)` on successful exit. This is what makes the "upgrade hashes in place" behavior observable.
- **File reads:** The delegate reads source files and computes hashes; with `git_aware` it may invoke `git` (subprocess) via the delegate.
- **No truth-value changes:** Viewing/assert/retract never happens here.

## Error Handling

- **Propagated, not swallowed:** There is no `try/except` in this function. Any `OSError` from `Path.resolve`, missing DB file errors from `Storage.load`, or exceptions raised by the delegate bubble up to the caller.
- **Missing repos:** If a `source` cannot be resolved to a path, the delegate is expected to handle it (typically by skipping or reporting the node as stale — see `tests/test_check_stale_issue25.py` for the specific behaviors around unresolved sources).
- **PG backend:** `_pg_dispatch` may raise if `PgApi` construction or method dispatch fails; nothing here tries to recover.

## Usage Patterns

Typical callers:

- CLI subcommand (e.g. `reasons check-stale`) — passes `db_path` and optionally `--upgrade-hashes` / `--git-aware`.
- Periodic staleness job — passes `repos={...}` for an explicit repo map, leaves flags off, reads `stale` list.
- Test harness — passes a temporary `db_path` and a controlled repo map (see `tests/test_check_stale.py`, `tests/test_check_stale_issue25.py`, `tests/test_pg_dispatch.py`).

Caller obligations:

- Do not mutate `net.repos` concurrently; if `repos` is passed explicitly it wins.
- Expect the DB to be written to only when the two flags are set — a read-only caller is safe.
- Follow up with `pin_sources` / `hash_sources` (both in this file) if you want the underlying hashes/pins to be established before calling `check_stale`.

## Dependencies

- **Internal:** `reasons.check_stale.check_stale` (delegate), `reasons.storage.Storage` + `_with_network` helper, `reasons.pg.PgApi` (via `_pg_dispatch`), and `reasons.Network` (via `_with_network`).
- **External/stdlib:** `pathlib.Path`. Indirectly `hashlib`, `subprocess` (git), `sqlite3` via the delegate.

## Assumptions not enforced by the type system

- `repos` is a flat `dict[str, str]` — the `Path(v)` conversion will raise if a value is not a str (e.g. someone passing a `Path` directly would be fine, but a `Path`-typed value in the signature would not actually be accepted at the call site of `Path(v)` — it happens to work, but the annotation says str).
- `net.repos` values are always strings; not enforced by the `Network` class as far as this call site knows.
- `db_path`'s parent exists — `Path(db_path).resolve().parent` does not create directories.
- `pg_conninfo` implies `domain_id` is meaningful; the signature permits `pg_conninfo` set with `domain_id=None`.
- The delegate guarantees the tuple order `(results, upgraded, sha_bumped)` — a contract by convention, not by type.
- `write=git_aware` assumes `pinned_sha` bumps are always safe writes; no explicit conflict-detection here.

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — The actual staleness algorithm: file reading, hash comparison, prefix-match upgrades, and git-log short-circuit for pinned SHAs.
- [function] `reasons/api.py:_with_network` — How the DB is loaded/cached and when the write-mode context triggers a save (essential for reasoning about `upgrade_hashes`/`git_aware` persistence).
- [function] `reasons/api.py:pin_sources` — The natural companion that establishes `pinned_sha` metadata so `git_aware=True` short-circuits become meaningful.
- [file] `tests/test_check_stale_issue25.py` — Documents edge cases the implementation is expected to handle (unresolved sources, truncated hashes, etc.).
- [function] `reasons/api.py:hash_sources` — Fills in missing `source_hash` values; without it, `check_stale`'s `checked` count will under-report.

## Beliefs

- `check-stale-delegates-to-check-stale-module` — The staleness algorithm lives in `reasons/check_stale.py`; `api.check_stale` only orchestrates lifecycle and reshaping.
- `check-stale-write-mode-gated-by-flags` — The network is opened with `write=True` only when `upgrade_hashes` or `git_aware` is set; otherwise no DB mutation can occur.
- `check-stale-falls-back-to-net-repos` — When `repos` is `None`, the function uses `net.repos` (the persisted repo map); passing an explicit dict overrides it.
- `check-stale-pg-dispatch-bypasses-sqlite-path` — If `pg_conninfo` is truthy, the function returns via `_pg_dispatch` and never touches `_with_network` or the local `check_stale` module.
- `check-stale-checked-count-is-eligibility-count` — `checked` counts IN nodes that have both `source` and `source_hash`, not the number of files actually opened.
