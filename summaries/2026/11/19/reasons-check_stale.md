---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/check_stale.py

# `reasons/check_stale.py` — Source Staleness Detection

## 1. Purpose

This module owns **tracking whether an `IN` node's cited source file has drifted** from the state it was created against. A node carries two fields relevant here: `source` (a string like `repo-name/path/file.md`) and `source_hash` (an SHA-256 of the source file at ingest time). `check_stale` compares the stored hash against the current file to flag stale nodes; a secondary, cheaper path uses git commit SHAs (`pinned_sha` / `pinned_lines`) to skip full-file hashing when nothing relevant changed.

It also provides the **write-side counterparts**: `hash_sources` (backfill missing hashes), `pin_sources`/`pin_update` (record git SHAs), and `pin_lines` (record a specific line range for finer-grained staleness).

A mirror lives at `reasons_lib/check_stale.py`; the authoritative one used by the package is `reasons/check_stale.py`.

## 2. Key Components

### Hashing helpers
- **`hash_file(path)`** — full 64-hex-char SHA-256 of the file bytes.

### Path resolution
- **`resolve_source_path(source, repos, db_dir, agent)`** — maps a source string to an absolute `Path`, trying in order:
  1. `repos[agent] / source` when `agent` is set (agent-imported beliefs whose source is relative to the agent's repo),
  2. `db_dir / source` (expert repos where sources live next to `reasons.db`),
  3. If the source has no `/`, treat it as a bare path relative to cwd,
  4. `repos[first_component] / rest`,
  5. `~/git/first_component/rest` fallback,
  returning `None` if nothing exists on disk.

### Staleness check (read path, but mutates nodes)
- **`check_stale(network, repos, db_dir, upgrade_hashes=False, git_aware=False) -> (results, upgraded, sha_bumped)`** — iterates all nodes sorted by id; only nodes with `truth_value == "IN"`, a `source`, and a `source_hash` are inspected. Returns three things:
  - `results`: list of dicts `{node_id, old_hash, new_hash, source, source_path, reason}` where `reason` ∈ `source_deleted | truncated_hash | content_changed`,
  - `upgraded`: count of legacy 16-char hashes expanded to full,
  - `sha_bumped`: count of `pinned_sha` advances performed in place.

### Backfill / pinning (write path)
- **`hash_sources(network, repos, force=False, db_dir=None)`** — sets `node.source_hash = hash_file(path)` for any node that has a source (and no hash, unless `force`). Returns `{node_id, source, hash, was_empty}` per node.
- **`pin_sources(network, ..., force=False, pin_urls=False)`** — for each `IN` node with a resolvable source and a git-committed SHA, writes `metadata["pinned_sha"]` and `metadata["verified_at"]`, backfills `source_hash` if empty, and optionally rewrites GitHub `blob/<branch>/…` URLs to `blob/<sha>/…` via `pin_source_url`.
- **`pin_update(network, node_ids, ...)`** — bumps `pinned_sha` to current HEAD for named nodes. Unlike `pin_sources`, it never silently skips: failures are returned as `{"node_id", "error": "not found" | "no source" | "source not found" | "not in git repo"}`.
- **`pin_lines(network, node_id, line_start, line_end, ...)`** — sets `metadata["pinned_lines"] = "start-end"`, auto-pinning `pinned_sha` and `source_hash` if not already present.

### Git plumbing
- **`get_file_commit_sha(filepath)`** — `git log -1 --format=%H -- <name>` with `cwd=filepath.parent`; `None` when not in a repo or on error.
- **`file_changed_since(filepath, since_sha)`** — `True` if commits touched the file `since_sha..HEAD` OR the working tree has staged/unstaged diffs vs HEAD. Returns `True` on any git error or timeout (deliberately conservative — see §7).
- **`file_lines_changed_since(filepath, since_sha, line_start, line_end)`** — single `git diff -U0 since_sha -- <name>`, then checks any hunk overlaps `[line_start, line_end]`.
- **`parse_diff_hunks(diff_output)`** — regex over `@@ -a[,b] +c[,d] @@`, returning **OLD-side** `(start, end)` ranges, skipping `count == 0` (pure-insertion hunks that don't touch old lines).
- **`pin_source_url(url, sha, source_path="")`** — rewrites `https://github.com/owner/repo/blob/<ref>/<path>` to `<sha>`; uses `source_path` to disambiguate when the branch name contains slashes.

## 3. Patterns

- **Procedural, no classes.** The module is a bag of free functions taking a `Network` plus lookup paths. This matches the rest of `reasons/`.
- **Report-list return convention.** Write functions return `list[dict]` of per-node outcomes rather than raising, so callers (CLI in `api.py`/`cli.py`) can print a summary. The only exceptions are `pin_lines` (raises `KeyError`/`ValueError` on misuse) and `check_stale` (returns tuples).
- **Cheap-then-expensive tiering.** With `git_aware=True`, a pinned SHA lets the code answer "did this file change?" with one `git log`; only if it did do we pay for `hash_file`.
- **Graceful degradation via truthy errors.** All git helpers return `True` on error/timeout, which drives callers into the content-hash path — safe over fast.
- **Prefix-upgrade for legacy data.** 16-char hashes are recognized as truncated and matched by prefix; `upgrade_hashes=True` lets callers rewrite them to full hashes inline.
- **In-place mutation of `Node`.** `check_stale`, `hash_sources`, and all `pin_*` functions mutate `node.source_hash` / `node.metadata` directly; the caller is responsible for persisting the network.

## 4. Dependencies

**Imports:** `hashlib`, `re`, `subprocess`, `datetime.date`, `pathlib.Path`, and `Network` from `.network`. It assumes the node type `network.nodes: dict[str, Node]` exposes `.truth_value`, `.source`, `.source_hash`, `.source_url`, `.metadata` (a mutable dict).

**Imported by:** `reasons/api.py`, `reasons/pg.py` (Postgres backing store), `reasons/verify.py`, the mirror `reasons_lib/check_stale.py`, and tests `test_check_stale.py`, `test_check_stale_issue25.py`, `test_pin_sources.py`.

## 5. Flow

`check_stale` control flow per node:

```
skip if truth_value != "IN"
skip if not source or not source_hash
path = resolve_source_path(...)
  └─ None → results.append(reason="source_deleted"); continue

if git_aware and pinned_sha:
    if pinned_lines:
        parse "ls-le"  (malformed → fall through)
        if NOT file_lines_changed_since(path, pinned_sha, ls, le):
            if newer sha: bump pinned_sha, verified_at=today, source_hash=hash_file
            continue
    elif NOT file_changed_since(path, pinned_sha):
        continue

current_hash = hash_file(path)
if current_hash != node.source_hash:
    if len(old)==16 and current.startswith(old):
        upgrade_hashes → mutate and continue     # no result recorded
        else → result reason="truncated_hash"
    else → result reason="content_changed"
elif git_aware and pinned_sha:
    new_sha = get_file_commit_sha(path)
    if new_sha != pinned_sha:
        bump pinned_sha + verified_at; sha_bumped += 1
```

Note the asymmetry: with `pinned_lines`, when the pinned lines are clean but the file has advanced, the pin **and `source_hash` are relocated to the new commit** (node is *not* marked stale). With whole-file pinning and `file_changed_since` returning `False`, we skip content hashing entirely without writing anything.

## 6. Invariants

- **Only `IN` nodes are stale-checkable.** `check_stale`/`pin_sources` skip `OUT`, `UNDECIDED`, etc.
- **Both `source` and `source_hash` are required** for `check_stale` to inspect a node; `hash_sources` only needs `source`.
- **`pinned_lines` format is `"<start>-<end>"`**, both parsable as ints. Malformed input silently falls through to whole-file hashing in `check_stale`, but `pin_lines` enforces `line_start >= 1` and `line_end >= line_start` on write.
- **Legacy hashes are exactly 16 hex chars**; only those participate in prefix-upgrade logic.
- **Diff hunks use the OLD side** — `parse_diff_hunks` returns ranges referencing `since_sha`, matching the stored `pinned_lines`, and pure-insertion hunks (`count == 0`) are dropped so inserting text above a range doesn't flag it.
- **`repos` defaults to `network.repos`** (converted from `dict[str,str]` to `dict[str,Path]`) when the argument is `None`.
- **Callers must persist the network** after `check_stale`/`hash_sources`/`pin_*` — mutations are in-place and nothing is saved here.

## 7. Error Handling

- **Git helpers are total.** `file_changed_since` and `file_lines_changed_since` catch `subprocess.TimeoutExpired` and treat any non-zero `returncode` as "changed" → callers fall back to hashing, biasing toward false stale over missed stale.
- **`get_file_commit_sha` swallows failure** and returns `None`; `pin_sources` quietly skips such nodes, while `pin_update` records `{"error": "not in git repo"}`.
- **Missing source files are not errors** — `check_stale` reports `reason="source_deleted"`; `hash_sources`/`pin_sources` skip.
- **`pin_lines` is the only raising function**: `ValueError` for bad line ranges or a node with no source, `KeyError` for unknown node id.
- **A subtle limitation:** git helpers pass `cwd=filepath.parent` and `filepath.name` as the path filter. That relies on git's repo discovery walking up from the parent; it works for files in a repo but means the subprocess always scopes the path filter to the immediate filename inside the parent directory (which is what `cwd`-relative resolution produces).

## Topics to Explore

- [file] `reasons/network.py` — Defines `Node`, `Network.nodes`, and the `repos`/`metadata` shape this module depends on
- [function] `reasons/verify.py:verify` — Downstream consumer that probably turns `check_stale` results into a report
- [file] `reasons/pg.py` — Postgres backing store that reuses these functions against stored nodes
- [file] `tests/test_check_stale_issue25.py` — Regression test likely covering the truncated-hash upgrade path
- [general] `git-aware-staleness` — How `pinned_sha` + `pinned_lines` combine to avoid full-file hashing on every run

## Beliefs

- `check-stale-processes-only-in-nodes` — `check_stale` and `pin_sources` skip any node whose `truth_value` is not `"IN"`.
- `git-errors-yield-changed-true` — `file_changed_since` and `file_lines_changed_since` return `True` on subprocess errors or timeouts, forcing content-hash fallback.
- `truncated-hashes-are-16-hex-chars` — Legacy truncated `source_hash` values are exactly 16 characters and are recognized by prefix match against the full 64-char hash.
- `check-stale-mutates-network-in-place` — `check_stale`, `hash_sources`, and all `pin_*` functions write to `node.source_hash` / `node.metadata` directly; the caller must persist the network.
- `resolve-source-path-prefers-agent-repo` — `resolve_source_path` resolution order is agent repo, then `db_dir`, then `repos[first_component]`, then `~/git/...`.
- `pin-lines-auto-pins-sha` — `pin_lines` sets `pinned_sha` and `source_hash` automatically when the node was not previously pinned.
