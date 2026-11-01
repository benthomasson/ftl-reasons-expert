---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/check_stale.py

# `reasons/check_stale.py`

## Purpose

This module owns the **staleness detection and source-pinning** lifecycle for belief nodes in the `reasons` network. A "belief" is a node whose truth is anchored to an external source file (a markdown doc, code snippet, etc.). Over time those source files drift. This file answers the question: *"Is the file a belief was extracted from still the same file it was when the belief was created?"*

It does two related things:

1. **Detect staleness** (`check_stale`, `hash_sources`) — compare stored SHA-256 hashes against current file content.
2. **Git-aware pinning** (`pin_sources`, `pin_update`, `pin_lines`) — record *which commit* a belief was sourced from, and use `git diff` to cheaply verify whether the relevant lines actually changed, avoiding a full content hash comparison.

This is a pure computation + mutation layer. It never I/Os to a database itself; it mutates `network.nodes` in place and returns result dicts for the caller to persist.

---

## Key Components

### Core Detection

| Function | Contract |
|---|---|
| `hash_file(path) → str` | Returns a 64-char hex SHA-256 of the file's bytes. Raises if the file doesn't exist (via `Path.read_bytes`). |
| `resolve_source_path(source, repos, db_dir, agent) → Path \| None` | Maps a source string (e.g. `"my-repo/docs/arch.md"`) to an absolute `Path`. Tries four strategies in order: agent repo → `db_dir` → `repos` dict → `~/git/<name>/`. Returns `None` if no candidate exists on disk. |
| `check_stale(network, repos, db_dir, upgrade_hashes, git_aware) → (list[dict], int, int)` | The main entry point. Iterates all **IN** nodes with both `source` and `source_hash` set. Returns `(stale_results, upgraded_count, sha_bumped_count)`. **Mutates** `node.source_hash`, `node.metadata` in place. Caller is responsible for saving. |
| `hash_sources(network, repos, force, db_dir) → list[dict]` | Backfills `source_hash` for nodes that have a `source` but no hash. With `force=True`, re-hashes all. Mutates `node.source_hash` in place. |

### Git-Aware Pinning

| Function | Contract |
|---|---|
| `get_file_commit_sha(filepath) → str \| None` | Runs `git log -1 --format=%H -- <filename>` in the file's parent directory. Returns the full 40-char SHA or `None` if not in a git repo. |
| `file_changed_since(filepath, since_sha) → bool` | Returns `True` if the file has any committed or uncommitted changes since `since_sha`. **Returns `True` on any git error or timeout** (fail-open → forces content-hash fallback). |
| `file_lines_changed_since(filepath, since_sha, line_start, line_end) → bool` | Same as above but scoped to a specific 1-based line range. Parses `git diff -U0` hunk headers and checks overlap. Also fail-open. |
| `parse_diff_hunks(diff_output) → list[tuple[int,int]]` | Extracts OLD-side `(start, end)` line ranges from unified diff `@@` headers. Skips hunks with count=0 (pure insertions). |
| `pin_sources(network, repos, db_dir, force, pin_urls) → list[dict]` | Pins all IN nodes to their current HEAD commit SHA. Stores `pinned_sha` + `verified_at` in metadata. Optionally rewrites `source_url` GitHub blob links to commit-SHA URLs. |
| `pin_update(network, node_ids, repos, db_dir) → list[dict]` | Bumps `pinned_sha` to current HEAD for a specific set of node IDs. Also refreshes `source_hash`. |
| `pin_lines(network, node_id, line_start, line_end, repos, db_dir) → dict` | Sets `pinned_lines` (stored as `"{start}-{end}"` string) on a single node. Auto-pins SHA if missing. Validates `line_start >= 1` and `line_end >= line_start`. |
| `pin_source_url(url, sha, source_path) → str` | Rewrites `https://github.com/user/repo/blob/<branch>/<path>` → `.../blob/<sha>/<path>`. |

---

## Patterns

- **Mutation-in-place with caller-saved state.** Every mutating function (`check_stale`, `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`) directly writes to `node.source_hash`, `node.metadata`, `node.source_url`. None of them call `network.save()`. The docstring on `check_stale` explicitly says *"caller must save the network."* This is consistent across the module.

- **Fail-open for git operations.** All three git-checking functions (`file_changed_since`, `file_lines_changed_since`, `get_file_commit_sha`) treat git errors, non-zero exit codes, and `subprocess.TimeoutExpired` as *"I can't tell, assume it changed."* This means the system degrades gracefully to full content hashing rather than silently marking a stale node as fresh.

- **Progressive path resolution.** `resolve_source_path` tries four increasingly generic strategies (agent-specific repo → db_dir → named repo → `~/git/` home fallback). This lets the same module work whether sources are in an agent's private repo, a shared expert repo next to `reasons.db`, or a standard `~/git` checkout.

- **Truncated-hash migration.** `check_stale` detects the legacy 16-char hash format: if `len(node.source_hash) == 16` and it's a prefix of the current full hash, it reports `"truncated_hash"` (or upgrades in place if `upgrade_hashes=True`). This is a one-time migration path, not a permanent dual-format.

- **Line-range pinning as a precision tier.** `pinned_lines` sits *on top of* `pinned_sha`. The check in `check_stale` only attempts the line-range fast path when `pinned_lines` is present; otherwise it falls back to the whole-file `file_changed_since`. This lets a belief cite "lines 42–58 of section-3.md" and skip hashing the rest of the file.

- **Sorted iteration for determinism.** Every loop over `network.nodes` uses `sorted(network.nodes.items())`, so stale reports and pin results are always in node-ID lexicographic order regardless of dict insertion order.

---

## Dependencies

**Imports (this file → others):**
- `hashlib` — SHA-256 hashing
- `re` — regex for diff hunk parsing and GitHub URL rewriting
- `subprocess` — shell out to `git log` / `git diff`
- `datetime.date` — `verified_at` timestamps
- `pathlib.Path` — filesystem operations
- `.network.Network` — the node graph structure (provides `network.nodes`, `network.repos`)

**Imported by (others → this file):**
- `reasons/api.py` — exposes `check_stale` / `pin_sources` as HTTP endpoints
- `reasons/pg.py` — PostgreSQL-backed variant likely calls these before/after DB writes
- `reasons/verify.py` — verification pipeline uses staleness as a gate
- `reasons_lib/check_stale.py` — library mirror (the `reasons_lib/` tree appears to be a vendored or generated copy)
- `tests/test_check_stale.py`, `tests/test_check_stale_issue25.py`, `tests/test_pin_sources.py` — unit tests

---

## Flow

### `check_stale` (the main path)

```
for each node (sorted by ID):
  1. Skip if truth_value != "IN"
  2. Skip if no source or no source_hash
  3. resolve_source_path(...)
       → None?  →  record "source_deleted", continue
  4. If git_aware AND node has pinned_sha:
       a. If pinned_lines present:
            → file_lines_changed_since(path, sha, ls, le)?
                NOT changed → bump sha if HEAD moved, continue (no stale)
                changed     → fall through to content hash
       b. Else (no line pinning):
            → file_changed_since(path, sha)?
                NOT changed → continue (no stale)
                changed     → fall through to content hash
  5. current_hash = hash_file(path)
  6. If current_hash != stored_hash:
       a. Truncated (16-char) prefix match?
            → upgrade_hashes?  upgrade in place, continue
            → else record "truncated_hash", continue
       b. Else record "content_changed"
  7. If hashes match but git_aware AND pinned_sha:
       → bump pinned_sha to current HEAD, record verified_at
```

### `pin_sources` (initial pinning)

```
for each IN node with a source (sorted):
  skip if already pinned (unless force=True)
  resolve path → get_file_commit_sha(path)
  set metadata["pinned_sha"], metadata["verified_at"]
  backfill source_hash if missing
  optionally rewrite source_url to commit SHA
```

### `pin_lines` (per-node line pinning)

```
validate line_start >= 1, line_end >= line_start
set metadata["pinned_lines"] = "{ls}-{le}"
if no pinned_sha yet:
    resolve path → get_file_commit_sha → set pinned_sha + verified_at + source_hash
return summary dict
```

---

## Invariants

1. **Only IN nodes are processed.** OUT, DEF, and other truth values are always skipped. This is checked at the top of every per-node loop (`node.truth_value != "IN"`).

2. **Line ranges are 1-based and contiguous.** `pin_lines` enforces `line_start >= 1` and `line_end >= line_start` with `ValueError`. The stored format is the string `"{start}-{end}"`.

3. **`pinned_lines` implies `pinned_sha`.** The line-range fast path in `check_stale` is only entered when `pinned_sha` exists. `pin_lines` auto-pins the SHA if missing. A node should never have `pinned_lines` without `pinned_sha` in practice, but the code handles the absence gracefully by skipping the fast path.

4. **Truncated hashes are exactly 16 hex chars.** The special-case check is `len(node.source_hash) == 16 and current_hash.startswith(node.source_hash)`. A 16-char hash that is *not* a prefix is treated as a genuine content change.

5. **Git checks are always fail-open.** If any git subprocess fails (non-zero exit, timeout, missing repo), the function returns `True` (meaning "changed" / "I can't verify"). The system never silently passes a node as fresh because git was unavailable.

6. **`source_hash` is always full SHA-256 (64 hex) after any write.** The only 16-char values are legacy stored values being read. `hash_file` always produces 64 chars.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Source file doesn't exist on disk | `resolve_source_path` returns `None` → recorded as `"source_deleted"` in results. No exception. |
| Not in a git repo / git command fails | `get_file_commit_sha` returns `None`; `file_changed_since` / `file_lines_changed_since` return `True` (assume changed). No exception propagates. |
| `subprocess.TimeoutExpired` (30s cap) | Caught in `file_changed_since` and `file_lines_changed_since`, returns `True`. `get_file_commit_sha` would propagate the exception (it has no try/except) — this is a latent gap. |
| Malformed `pinned_lines` (e.g. `"abc-def"`) | `int()` raises `ValueError`, caught by `except (ValueError, IndexError)`, falls through to full content-hash comparison. |
| Node not found in `pin_update` / `pin_lines` | `pin_update` records `{"error": "not found"}` in the results list. `pin_lines` raises `KeyError`. |
| `line_start < 1` or `line_end < line_start` | `pin_lines` raises `ValueError` with a descriptive message. |
| Non-GitHub URL in `pin_source_url` | Regex doesn't match → returns the URL unchanged. |

**Notable asymmetry:** `get_file_commit_sha` does *not* catch `TimeoutExpired` or general exceptions, unlike its two siblings. If git hangs in that specific call, the exception will propagate up through `check_stale` or `pin_sources` and crash the caller. This is likely a bug or an oversight.

---

## Topics to Explore

- [file] `reasons/network.py` — Defines the `Network` and node structure (`truth_value`, `source`, `source_hash`, `metadata`, `repos`) that this module reads and mutates; understanding the node schema is prerequisite to understanding all the guard clauses here.
- [file] `reasons/verify.py` — A consumer of `check_stale`; shows how staleness results feed into the verification pipeline and what happens to nodes flagged as stale (are they auto-defeated? queued for review?).
- [function] `reasons/check_stale.py:resolve_source_path` — The four-strategy path resolution is the single most fragile function in this module; understanding the agent/db_dir/repos/~/git hierarchy requires reading the import and agent-sync code.
- [file] `tests/test_check_stale_issue25.py` — The filename suggests a specific bug fix; comparing it with `test_check_stale.py` reveals edge cases (likely around truncated hashes or git-fail-open behavior) that the general test file doesn't cover.
- [general] `reasons-lib-duplication` — Both `reasons/` and `reasons_lib/` contain a `check_stale.py`; understanding whether one is generated from the other, or whether they can drift, affects how confidently you can modify one without checking the other.

---

## Beliefs

- `check-stale-mutates-network-in-place` — All mutating functions in this module (`check_stale`, `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`) write directly to `node.source_hash`, `node.metadata`, and `node.source_url` without saving; the caller is contractually responsible for persisting the network.
- `git-errors-fail-open-to-content-hash` — Every git-based staleness check (`file_changed_since`, `file_lines_changed_since`) returns `True` on git failure or timeout, ensuring the system falls back to full SHA-256 content comparison rather than silently marking a node as fresh.
- `pinned-lines-requires-pinned-sha` — The line-range fast path in `check_stale` is only entered when `node.metadata["pinned_sha"]` is set; `pin_lines` auto-pins the SHA if missing, so a node with `pinned_lines` but no `pinned_sha` will silently skip the line-range optimization and fall through to whole-file hashing.
- `truncated-hash-detection-is-16-char-exact` — The legacy-hash migration path triggers only when `len(source_hash) == 16` AND it is a prefix of the current 64-char hash; a 16-char hash that doesn't prefix-match is classified as `"content_changed"`, not `"truncated_hash"`.
- `get-file-commit-sha-lacks-timeout-guard` — Unlike `file_changed_since` and `file_lines_changed_since`, `get_file_commit_sha` does not catch `subprocess.TimeoutExpired`, so a hung git process in that call will propagate an unhandled exception through `check_stale` or `pin_sources`.
