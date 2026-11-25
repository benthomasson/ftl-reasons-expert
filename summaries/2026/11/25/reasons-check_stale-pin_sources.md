---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: pin_sources in reasons/check_stale.py

## `pin_sources` — Explanation

### 1. Purpose

`pin_sources` is the "freeze" operation for source-backed `IN` nodes. It records the **current last-commit SHA** of each node's source file into `node.metadata["pinned_sha"]` (plus `verified_at`), and backfills `source_hash` if missing. Existence of a `pinned_sha` is what enables the **git-aware** fast path in `check_stale`: instead of hashing content every pass, `check_stale` can ask git "did anything touch these lines since this SHA?" and defer content hashing entirely.

It's the bulk counterpart to `pin_update` (per-node re-pin) and `pin_lines` (line-range pin). Typically exposed as a CLI subcommand (`pin-sources`).

### 2. Contract

**Preconditions (implicit, not type-enforced):**
- `network.nodes[*].metadata` is a **dict** for every node with a `source`. The function does `node.metadata.get("pinned_sha")` and later `node.metadata["pinned_sha"] = sha` without a `None` guard. `pin_lines` handles the `None` case by assigning `{}` first; `pin_sources` does **not** — a `None` metadata will raise `AttributeError`.
- `node.source`, when present, resolves under one of the search locations of `resolve_source_path`.
- The file lives inside a git working tree for its SHA to be resolvable.

**Postconditions (for each processed node):**
- `node.metadata["pinned_sha"]` equals the most recent commit SHA touching the file.
- `node.metadata["verified_at"]` equals `date.today().isoformat()`.
- `node.source_hash` is populated if it was previously empty (never overwritten otherwise).
- If `pin_urls=True` and `node.source_url` is a GitHub blob URL, `node.source_url` is rewritten to reference the pinned SHA.
- Caller **must persist** the network — mutations are in-memory only.

**Invariants:**
- Only nodes with `truth_value == "IN"` and a non-empty `source` are ever considered.
- A node that already has a truthy `pinned_sha` is skipped unless `force=True`.
- `verified_at` is *last* pin time, not first — it is overwritten on every successful pin.

### 3. Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `network` | `Network` | The graph to mutate. Nodes must have `truth_value`, `source`, `metadata`, `source_hash`, optionally `source_url`. | Called for side effects on `network.nodes`. |
| `repos` | `dict[str, Path] \| None` | Map of repo-name → filesystem path, used by `resolve_source_path`. | `None` → falls back to `network.repos` (string values coerced to `Path`). Empty dict is honored (no fallback). |
| `db_dir` | `Path \| None` | Directory of `reasons.db`; used as a resolution root for expert-repo sources. | `None` → skipped in resolution. |
| `force` | `bool` | Re-pin even nodes that already have `pinned_sha`. | With `force=False`, the function effectively becomes "pin-new-only." With `force=True`, `was_empty` in results becomes meaningful. |
| `pin_urls` | `bool` | Rewrite `node.source_url` from branch ref to SHA via `pin_source_url`. | Only affects `https://github.com/<owner>/<repo>/blob/...` URLs; anything else is a no-op. `source_url` missing/empty → skipped. |

### 4. Return Value

A `list[dict]`, one entry per successfully pinned node:

```python
{"node_id": str, "source": str, "pinned_sha": str, "was_empty": bool}
```

- `was_empty` is `not node.metadata.get("pinned_sha")` **before** assignment. Under `force=False`, this is **always** `True` (the guard filters out pre-pinned nodes anyway). Only with `force=True` will it ever be `False`.
- Nodes that were skipped (already pinned, no source, non-`IN`, unresolved path, not in git) produce **no entry** — the return value only reports successes. Callers wanting diagnostics must diff inputs against returned `node_id`s.
- The returned dict mirrors what was written; the caller does not need to inspect the node again.

### 5. Algorithm

1. **Resolve repos:** if `repos is None and network.repos`, rebuild `{k: Path(v)}` from `network.repos`.
2. **Snapshot `today`** as ISO date string.
3. **Iterate `sorted(network.nodes.items())`:** deterministic order (by node-id).
4. **Filters (each `continue`s):**
   - `node.truth_value != "IN"`
   - not `node.source`
   - not `force` **and** `node.metadata.get("pinned_sha")` is truthy
5. **Resolve path:** `resolve_source_path(node.source, repos, db_dir, agent=…)`. `agent` comes from `node.metadata` if present. `None` → skip.
6. **Get SHA:** `get_file_commit_sha(path)` — a `git log -1` run in `path.parent`. `None` (not in git, or errors) → skip.
7. **Compute `was_empty`** from the pre-existing `pinned_sha`.
8. **Write metadata:** `pinned_sha = sha`, `verified_at = today`.
9. **Backfill `source_hash`** only if it is falsy.
10. **Optional URL rewrite:** `if pin_urls and node.source_url:` → `pin_source_url(url, sha, node.source)`.
11. **Append result dict.**

### 6. Side Effects

- **Mutates `node.metadata`**: adds/overwrites `pinned_sha` and `verified_at`.
- **Mutates `node.source_hash`** (backfill only — will not overwrite an existing value; `hash_sources(force=True)` is the tool for that).
- **Mutates `node.source_url`** when `pin_urls=True` and URL is GitHub-shaped.
- **Filesystem I/O:** `hash_file` reads the entire source file's bytes.
- **Subprocess:** one `git log -1 --format=%H -- <name>` call per candidate, with `cwd=path.parent`, `timeout=30`.
- **Not persisted automatically.** Without a subsequent `network.save()`, nothing on disk changes.

### 7. Error Handling

- **`get_file_commit_sha` swallows** non-zero returncode and empty stdout → returns `None` → node silently skipped. This is expected for files outside any git repo.
- **`subprocess.TimeoutExpired`** is **not** caught in `get_file_commit_sha` (unlike `file_changed_since` / `file_lines_changed_since`, which catch it and return `True`). A hung git invocation aborts `pin_sources`.
- **`FileNotFoundError`** if `git` isn't on `PATH` → propagates.
- **`OSError` from `hash_file`** (permissions, vanished file after resolution) → propagates.
- **`AttributeError`** if a node with a `source` has `metadata is None` — the code assumes it's a dict. Contrast with `pin_lines`, which defensively initializes it.
- **`pin_source_url` never raises**; it returns the input unchanged if the regex doesn't match.
- **No partial-failure tracking:** nodes processed before an exception have already been mutated in memory; if the caller catches and discards the exceptions, the network is left in a half-pinned state.

### 8. Usage Patterns

- Invoked as a bulk operation before enabling `git_aware=True` in `check_stale`, so subsequent runs can skip content hashing on unchanged files.
- Typical caller: a CLI handler that (a) loads the network, (b) calls `pin_sources`, (c) persists the network, (d) prints `results`. **Persisting the network is a caller obligation.**
- To re-pin already-pinned nodes to HEAD, callers either pass `force=True` here or use `pin_update` for the selective case.
- To sync `source_hash` with the pinned SHA after a legitimate content change, callers combine this with `hash_sources(force=True)` (not called from within `pin_sources`).

### 9. Dependencies

- **Stdlib:** `subprocess`, `hashlib`, `datetime.date`, `pathlib.Path`.
- **Internal module:** `.network.Network` (only for the `repos` fallback attribute).
- **Sibling helpers in this module:**
  - `resolve_source_path` — multi-root source resolution.
  - `get_file_commit_sha` — git wrapper.
  - `hash_file` — SHA-256 backfill.
  - `pin_source_url` — GitHub blob URL SHA rewrite.

### Assumptions worth flagging

- **`node.metadata` is a dict** whenever the node has a `source`. Not enforced by `Network` typing; `pin_lines` treats it as possibly `None`.
- **`network.repos` values are stringifiable**; they get wrapped in `Path`.
- **`resolve_source_path` returns a real file**, so `hash_file` won't `FileNotFoundError` — but a race is possible.
- **`get_file_commit_sha` treats "any other git error" the same as "file not in git"** (both → `None` → skip), so a transient git failure is silently absorbed and the node stays unpinned.

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — Consumes `pinned_sha` (`git_aware=True`) to avoid content hashing; shows what pinning is *for*.
- [function] `reasons/check_stale.py:resolve_source_path` — Multi-root search order (agent → db_dir → repos → `~/git/`) that determines which file gets pinned.
- [function] `reasons/check_stale.py:pin_source_url` — URL rewriting logic; only GitHub blob URLs are handled, brand-name-agnostic callers may be surprised.
- [file] `tests/test_pin_sources.py` — Encodes expected behavior around `force`, `was_empty`, and URL pinning.
- [general] `network-metadata-nullability` — Whether `node.metadata` can ever be `None` given `pin_sources`'s unguarded `.get`; determines whether the AttributeError path is reachable.

## Beliefs

- `pin-sources-mutates-in-place` — `pin_sources` returns a summary list but the real effect is in-place mutation of `node.metadata`, `node.source_hash`, and optionally `node.source_url`; callers must persist `network` themselves.
- `pin-sources-skips-pinned-unless-force` — Nodes with a truthy existing `pinned_sha` are skipped unless `force=True`; consequently `was_empty` in the return value is always `True` when `force=False`.
- `pin-sources-requires-dict-metadata` — The implementation assumes `node.metadata` is a dict for any node with a `source`, and will raise `AttributeError` if it is `None` (unlike `pin_lines`, which defensively initializes it).
- `pin-sources-absorbs-non-git-and-git-errors` — `get_file_commit_sha` returns `None` for both "not in a git repo" and "git command failed", causing those nodes to be silently skipped rather than reported.
- `pin-sources-backfills-hash-without-overwriting` — `source_hash` is only written when previously falsy; refreshing a stale hash requires a separate `hash_sources(force=True)` call.
- `pin-urls-rewrites-only-github-blobs` — `pin_source_url` only rewrites URLs matching `https://github.com/<owner>/<repo>/blob/...`; all other URL shapes are returned unchanged.
