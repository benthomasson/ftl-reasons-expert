---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: pin_lines in reasons/check_stale.py

# `pin_lines` — Explanation

## Purpose

`pin_lines` sets a *line-range pin* (`node.metadata["pinned_lines"]`) on a specific node, and opportunistically *auto-pins* a git commit SHA (`node.metadata["pinned_sha"]`) if the node doesn't already have one. It is the entry point for "this node's belief came from lines X–Y of its source file," which is then consumed by `check_stale` (via `file_lines_changed_since`) so that staleness is only triggered when the diff hunks overlap the pinned range — not when any unrelated part of the file changes.

It exists alongside the other Git-aware pinning helpers (`pin_sources`, `pin_update`) in the same module, and is the per-line variant of `pin_sources`.

## Contract

**Preconditions (enforced by raising):**
- `line_start >= 1`
- `line_end >= line_start`
- `node_id` must be a key in `network.nodes`
- the resolved node must have a non-empty `node.source`

**Postconditions:**
- `node.metadata["pinned_lines"]` is set to the string `f"{line_start}-{line_end}"` (always, regardless of anything else that happens later).
- If the node had no `pinned_sha` **and** the source path could be resolved **and** git returned a SHA: `pinned_sha` and `verified_at` are set, and `auto_pinned == True` in the return value.
- If the node had no `pinned_sha` **and** the source path could be resolved (even without a git SHA): `source_hash` is recomputed via `hash_file`.
- If the node already had a `pinned_sha`, nothing else is touched (no `source_hash` refresh, no re-pin).

**Invariants assumed but not enforced:**
- `node.metadata` is a mutable `dict` (falsy values are replaced with `{}`).
- The line range refers to the file *at* `pinned_sha`, not to the current working tree — that's the whole concept `file_lines_changed_since` relies on.
- The network is not persisted here; the caller is responsible for saving.

## Parameters

| Param | Type | Meaning |
|---|---|---|
| `network` | `Network` | Container holding `.nodes` (dict keyed by id) and `.repos` (map name → path or str). |
| `node_id` | `str` | Key into `network.nodes`. Missing → `KeyError`. |
| `line_start` | `int` | 1-based start line, inclusive. `< 1` → `ValueError`. |
| `line_end` | `int` | 1-based end line, inclusive. `< line_start` → `ValueError`. |
| `repos` | `dict[str, Path] \| None` | Optional override of repo mapping. If `None`, falls back to `network.repos` cast to `Path`. |
| `db_dir` | `Path \| None` | Passed through to `resolve_source_path` as the "sources live next to reasons.db" fallback. |

**Edge cases** the types don't capture:
- `network.repos` may be `dict[str, str]` (str paths); the fallback casts them to `Path`.
- `line_start == line_end` is legal (single-line pin).
- Node with a `source` that resolves to a local file but the file is *not* inside a git repo → `get_file_commit_sha` returns `None`; `source_hash` is still set; `pinned_sha` stays as-is (possibly empty).

## Return Value

A plain dict:

```python
{
  "node_id": str,
  "pinned_lines": "start-end",       # always present
  "pinned_sha": str,                 # "" if not set
  "auto_pinned": bool,               # True only if pin_lines itself just set it
}
```

`pinned_sha` may be `""` — callers must handle a missing SHA (e.g., non-git source). `auto_pinned` distinguishes "SHA was set by this call" from "SHA was already present."

## Algorithm

1. **Validate bounds** — raise `ValueError` for `line_start < 1` or `line_end < line_start`.
2. **Look up node** — `network.nodes.get(node_id)`; missing → `KeyError`.
3. **Require a source** — falsy `node.source` → `ValueError`.
4. **Resolve repo map** — if `repos is None`, use `network.repos` (cast to `Path`).
5. **Ensure metadata dict** — `if not node.metadata: node.metadata = {}` (covers `None` and empty dict).
6. **Write `pinned_lines`** — always, before any git work. This is intentional: the range pin is cheap and does not depend on git.
7. **If no existing `pinned_sha`:**
   - a. Read `agent` from metadata.
   - b. `resolve_source_path(...)` — `None` means "cannot find file" → skip auto-pin block entirely (but `pinned_lines` is already set).
   - c. `get_file_commit_sha(path)` — `None` means not a git repo → skip SHA write.
   - d. If SHA: set `pinned_sha`, set `verified_at = today`, mark `auto_pinned = True`.
   - e. **Unconditionally** (whether or not a SHA was found) recompute `node.source_hash = hash_file(path)`. This is the only line that runs even when `sha is None`.
8. **Return** the summary dict.

## Side Effects

- **In-memory mutations** (no persistence):
  - `node.metadata["pinned_lines"]`
  - conditionally `node.metadata["pinned_sha"]`, `node.metadata["verified_at"]`
  - conditionally `node.source_hash`
- **Filesystem I/O**:
  - `hash_file` reads the entire source file with `path.read_bytes()`.
- **Subprocess**:
  - `get_file_commit_sha` runs `git log -1 --format=%H -- <basename>` with `cwd=filepath.parent`, timeout 30s.
- **Not atomic / no rollback**: `pinned_lines` is written before any of the git/hash work. If `get_file_commit_sha` (subprocess error) or `hash_file` raises, the metadata is left in a half-mutated state and the exception propagates.

## Error Handling

**Raised (caller must handle):**
- `ValueError` — bad `line_start`/`line_end`, or node has no `source`.
- `KeyError` — `node_id` not present in `network.nodes`.
- `OSError` (and subclasses) — from `path.read_bytes()` in `hash_file` or `subprocess.run` if the working directory does not exist.
- `subprocess.TimeoutExpired` — if `git log` takes >30s. **Not caught here** (unlike `file_changed_since` / `file_lines_changed_since`, which return `True` on timeout to force fallback).

**Swallowed / degraded:**
- `git log` non-zero exit or empty output → `get_file_commit_sha` returns `None`; no exception.
- Unresolvable source path → `path is None`; the SHA/hash block is skipped silently.

Notably, this function is *not* error-tolerant to the same degree as the reader-side helpers: it will raise on `subprocess.TimeoutExpired` and on file-read errors, whereas `file_lines_changed_since` returns `True`.

## Usage Patterns

- Typically invoked from a CLI/admin tool **after** `pin_sources` has already pinned a file-level SHA, to narrow a node to specific lines.
- Or invoked alone; it will auto-pin the SHA as a convenience (hence `auto_pinned` in the return, likely used to log "SHA was newly set" or to decide whether to emit a save).
- **Caller obligation:** persist the network. Mutations are in-memory only; the function returns a summary dict, not a saved state.
- **Caller obligation:** interpret `pinned_sha == ""` as "no git pin available" — subsequent `check_stale` calls with `git_aware=True` will fall back to full content hashing for this node.
- The returned `pinned_lines` string is the exact format `check_stale` parses with `parts = pinned_lines.split("-"); int(parts[0]), int(parts[1])`.

## Dependencies

- `reasons.network.Network` — parameter type; only `.nodes` and `.repos` are used.
- Same-module helpers: `resolve_source_path`, `get_file_commit_sha`, `hash_file`.
- Stdlib: `hashlib`, `subprocess`, `datetime.date`, `pathlib.Path`.

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — Consumes `pinned_lines` via `file_lines_changed_since` to skip content hashing when hunks don't overlap the pinned range.
- [function] `reasons/check_stale.py:resolve_source_path` — The three-stage path resolution (agent repo → `db_dir` → repo map → `~/git/`) that determines whether auto-pin can even execute.
- [function] `reasons/check_stale.py:parse_diff_hunks` — Parses `git diff @@` headers into old-side ranges; the correctness of line-pinned staleness hinges on this.
- [file] `tests/test_check_stale.py` — Covers how `pinned_lines` interacts with the staleness detector; likely the best place to see expected pre/post states.
- [general] `git-aware-pinning` — The three-tier pinning model (`pinned_sha`, `pinned_lines`, `source_hash`) and how each degrades to the next when unavailable.

## Beliefs

- `pin-lines-requires-node-exists` — `pin_lines` raises `KeyError` when `node_id` is not in `network.nodes`, and `ValueError` when the resolved node has no `source`.
- `pin-lines-sets-pinned-lines-before-git-work` — `node.metadata["pinned_lines"]` is written before any git or hashing work, so it survives subsequent exceptions but leaves partial state on failure.
- `pin-lines-only-auto-pins-when-absent` — The auto-pin block is gated on `not node.metadata.get("pinned_sha")`; an existing SHA is never refreshed by this function (that's `pin_update`'s job).
- `pin-lines-sets-source-hash-without-sha` — `node.source_hash = hash_file(path)` runs whenever the path resolves, even if `get_file_commit_sha` returned `None`, producing a node with a hash but no pinned SHA.
- `pin-lines-does-not-persist` — The function mutates the in-memory node and returns a summary dict but writes nothing to disk; the caller must save the network.
