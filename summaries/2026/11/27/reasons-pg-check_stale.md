---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: check_stale in reasons/pg.py

# `PgApi.check_stale` — Explanation

## Purpose

`check_stale` scans every **IN** belief in the PostgreSQL-backed network that has a recorded `source` file path and a stored `source_hash`, re-hashes the on-disk file, and reports which beliefs have drifted from their source artifact. It is the PostgreSQL-native counterpart to the file-backed `Network.check_stale` — it does not load the whole network into memory, but instead runs one SQL query and streams results per node.

It exists because beliefs reference code/docs by `source` path; if the file changes out-of-band the belief is no longer grounded in that source. The method also handles one historical data-migration concern: older beliefs stored only the first 16 hex characters of the SHA-256 hash, so it can optionally promote those to full hashes.

## Contract

**Preconditions**
- `self.conn` is an open psycopg connection; `self.domain_id` is set.
- The `rms_nodes` table exists with `id, domain_id, truth_value, source, source_hash, metadata` columns.
- `repos`, if provided, is a `dict[str, str]` of `{repo_name: absolute_path}`.
- `hash_file` and `resolve_source_path` are importable from `reasons.check_stale`.

**Postconditions**
- Returns a summary dict; never mutates a belief's `truth_value`.
- If and only if `upgrade_hashes=True` and at least one 16-char truncated hash matches a full current hash, the corresponding `rms_nodes.source_hash` rows are `UPDATE`d and the transaction is `COMMIT`ted.
- Stale nodes (path missing, content changed, truncated-without-upgrade) are included in `results`; upgraded nodes are **not** included in `stale` but are counted in `upgraded`.

**Invariants**
- Only rows with `truth_value = 'IN'`, non-empty/non-null `source`, and non-empty/non-null `source_hash` are considered.
- Within a single call, one node's failure is *not* isolated — an exception aborts the loop.

## Parameters

| Param | Type | Meaning |
|---|---|---|
| `repos` | `dict[str, str] \| None` | Overrides the repo registry used for path resolution. Names map to filesystem roots. Converted to `Path` internally. If `None`/empty, `self._load_repos()` loads from `rms_network_meta` via `list_repos()`, which may itself return `None`. |
| `upgrade_hashes` | `bool` | If `True`, nodes whose stored hash is a 16-char prefix of the current file hash are repaired in place (written back to `source_hash`) and omitted from the stale report. If `False` (default), they are reported as `reason="truncated_hash"`. |
| `git_aware` | `bool` | **Accepted but unused.** No reference appears in the body. A caller passing this gets no behavioral change. |

Edge cases:
- `repos={}` is falsy and therefore falls back to `_load_repos()`; you cannot force "no repos" via this param.
- `meta` from the JSONB column may arrive as a `dict` (psycopg default) *or* a `str`; the code normalizes both, and treats `None`/`{}` as "no agent metadata."
- `source_hash` of any length ≠ 16 is always treated as a full hash — there is no schema constraint enforcing a hash length, so a 40-char hash is compared via straight equality.

## Return Value

```python
{
    "stale": [
        {
            "node_id": str,
            "old_hash": str,
            "new_hash": str | None,     # None only for source_deleted
            "source": str,
            "source_path": str | None,  # None only for source_deleted
            "reason": "source_deleted" | "truncated_hash" | "content_changed",
        },
        ...
    ],
    "checked": int,       # len(rows) — total IN nodes with source+hash
    "stale_count": int,   # len(stale) — upgraded nodes are NOT counted here
    "upgraded": int,      # number of hashes repaired in place
    "sha_bumped": 0,      # hard-coded
}
```

Callers must handle:
- `new_hash is None` (means the file is gone; not "empty file").
- `source_path is None` (means resolution failed — repo not registered, agent repo missing, etc.).
- `stale` entries where `reason == "truncated_hash"` mean the belief is *not* really stale; promotion was simply not requested.
- `sha_bumped` is a placeholder — always 0; no git-blame/commit SHA tracking is implemented, despite the `git_aware` parameter suggesting otherwise.

## Algorithm

1. **Resolve repo roots.** If `repos` is truthy, convert each value to `Path`. Else `self._load_repos()` queries `rms_network_meta` for `repo:*` rows and returns a `Path` dict (or `None`).
2. **Fetch candidate rows.** SQL selects `(id, source, source_hash, metadata)` from `rms_nodes` filtered by `domain_id`, `truth_value='IN'`, non-empty `source`, non-empty `source_hash`. Rows are fetched with a fresh cursor and are then sorted by `id` in Python (SQL provides no `ORDER BY`).
3. **Per row:**
   - Normalize `meta` (str → dict; empty → `{}`), extract `meta.get("agent")`.
   - `resolve_source_path(source, repo_paths, agent=agent)` → `Path | None`.
   - If `None`: append `{reason: "source_deleted", new_hash: None, source_path: None}` and `continue`.
   - `current_hash = hash_file(path)`.
   - If `current_hash == source_hash`: nothing to report.
   - Else if `len(source_hash) == 16` **and** `current_hash.startswith(source_hash)`:
     - if `upgrade_hashes`: queue `(current_hash, nid)` for update, `upgraded += 1`, `continue`.
     - else: append `{reason: "truncated_hash", ...}` and `continue`.
   - Else: append `{reason: "content_changed", ...}`.
4. **Commit upgrades.** If `upgrades_to_commit`, open a new cursor, `UPDATE rms_nodes SET source_hash = %s WHERE id = %s AND domain_id = %s` for each, then `self.conn.commit()`.
5. **Return the summary dict.**

Note that if no upgrades are queued, the connection is left in whatever transaction state the caller had — the earlier `SELECT` did not mutate anything, so no commit is needed for read-only runs.

## Side Effects

- **Reads from DB**: one `SELECT` on `rms_nodes`; possibly one `SELECT` on `rms_network_meta` via `_load_repos → list_repos`.
- **Reads from filesystem**: `resolve_source_path` may stat paths; `hash_file` reads the entire file to hash it.
- **Writes to DB**: conditional `UPDATE` on `rms_nodes.source_hash`, followed by `COMMIT`, only when `upgrade_hashes=True` and truncated matches are found.
- **No logging, no HTTP, no filesystem writes.**

## Error Handling

There is essentially **no error handling**:

- `hash_file` exceptions (file deleted between resolution and read, permission denied, `IOError`, etc.) propagate to the caller and abort the loop, leaving the whole check unfinished. There is no per-node `try/except`; even one unreadable file kills the method.
- `resolve_source_path` exceptions propagate similarly.
- DB errors (constraint violations on `UPDATE`, cursor errors) propagate.
- No `rollback()` is performed on exception — a partially applied batch of hash upgrades could conceivably be committed or rolled back depending on the caller's connection policy. Since the upgrades are all performed in one cursor within one implicit transaction, a mid-loop DB exception would leave the transaction uncommitted; the caller must roll back.
- `meta` being a non-`dict`, non-`str` type (e.g. `None`) is handled: `agent = None`.
- A `source` file that resolves to a directory is *not* detected — `hash_file` would raise.

## Usage Patterns

Typical caller: a CLI verb (something like `stale` or `check`) that iterates results and prints report entries. There is no batching / pagination — a large network is fully scanned per call.

- Default call: `api.check_stale()` — compares hashes, reports stale; safe to run against a live DB and does not commit anything.
- Migration call: `api.check_stale(upgrade_hashes=True)` — repairs 16-char truncated hashes. Callers must understand the returned `upgraded` count reflects *successful* repairs, and the repairs were committed before the method returned.
- `repos={'myrepo': '/abs/path'}` skips DB repo lookup — useful when running against a checkout the DB doesn't know about.
- The `git_aware` kwarg is a no-op; callers should not rely on git-commit tracking to gate anything.

Caller obligations:
- Handle `new_hash=None` and `source_path=None`.
- Treat `truncated_hash` as a "not really stale" signal; use `upgrade_hashes=True` to silence it.
- Roll back `self.conn` if this method raised mid-loop and the caller was sharing a transaction.

## Dependencies

- `.check_stale.hash_file` — responsible for the actual hashing; not inspected here.
- `.check_stale.resolve_source_path` — resolves `(source, repo_paths, agent)` to a `Path` or `None`.
- `pathlib.Path` — local import.
- `self._load_repos()` — DB repo registry loader; returns `dict[str, Path] | None`.
- `psycopg` via `self.conn.cursor()`.
- `json` (module-level) — for parsing JSONB `metadata` that arrives as a string.

## Assumptions Not Enforced by Types

1. **16 characters means "truncated SHA-256."** Nothing in the schema enforces this; a user could legitimately store a 16-char hash (e.g., for a different algorithm), in which case the code will silently treat a coincidental prefix match as a truncation and offer to upgrade it to the full hash.
2. **`current_hash` is longer than `source_hash` and is a full hash.** The `startswith` check assumes `hash_file` returns a full-length hex digest.
3. **`meta` is safe to index with `"agent"`** after str-parsing — assumed dict-shaped if non-falsy.
4. **`repos` param values are strings** — the code calls `Path(v)` without checking; a `Path` value would also work, but a non-string/non-Path raises.
5. **`resolve_source_path` handles `repo_paths=None`** — the code passes it through, so that helper is responsible for that `None` case.
6. **`rms_nodes` is the only source of truth for the node list** — the method trusts the DB and never verifies that a resolved path still exists before calling `hash_file` (file existence is inferred from `hash_file` succeeding).
7. **`git_aware` and `sha_bumped` are meant for future features** — they are dead surface area right now; a bot returning `{"sha_bumped": 0}` is always correct but meaningless.

## Topics to Explore

- [file] `reasons/check_stale.py` — Where `hash_file` and `resolve_source_path` actually implement path resolution (agent repos, repo-name prefix handling) and hashing; the assumptions above live here.
- [function] `reasons/pg.py:hash_sources` — Sibling routine that backfills `source_hash` for nodes whose value is empty; together with `check_stale(upgrade_hashes=True)` it forms the full source-hash lifecycle.
- [function] `reasons/pg.py:_load_repos` — Shows how the repo registry is loaded from `rms_network_meta`, and why it can return `None`.
- [file] `tests/test_check_stale_issue25.py` — Likely the regression test for the truncated-hash upgrade path; useful to confirm expected reporting semantics.
- [general] `non-pg-check-stale-comparison` — Compare with the file-backed `Network.check_stale` variant to understand intentional behavior drift (which aggregates same outputs, which adds git awareness, etc.).

## Beliefs

- `check-stale-excludes-out-nodes` — The SQL filter restricts the scan to `truth_value = 'IN'`; OUT nodes are never checked for staleness.
- `check-stale-upgrade-hashes-requires-16-char-stored-hash` — The in-place upgrade path fires only when `len(source_hash) == 16` and `current_hash.startswith(source_hash)`; longer stored hashes never take this branch.
- `check-stale-does-not-catch-file-errors` — File hashing errors from `hash_file` propagate to the caller; there is no per-node try/except.
- `check-stale-upgraded-nodes-excluded-from-stale-count` — When `upgrade_hashes=True`, nodes repaired in place do not appear in `stale` and are not counted in `stale_count`; they are counted only in `upgraded`.
- `check-stale-commits-upgrades` — When upgrades are queued, the method issues `UPDATE rms_nodes SET source_hash = ...` and calls `self.conn.commit()` before returning.
- `check-stale-git-aware-unused` — The `git_aware` parameter is accepted but never referenced in the function body.
- `check-stale-sha-bumped-hardcoded-zero` — The returned `sha_bumped` field is hard-coded to `0` on every call.
