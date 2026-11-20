---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: check_stale in reasons/check_stale.py

## `check_stale` — Stale Node Detector

### 1. Purpose

`check_stale` walks every **IN** node in a `Network` and determines whether the node's stored `source_hash` still matches the SHA-256 of the file it was sourced from. It exists because nodes in this system are *beliefs derived from external files* — if the underlying file changed, the belief may be out of date.

It is the read/verify half of the source-tracking subsystem (the write half being `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`). It does **not** persist anything — the caller is responsible for saving the network after mutation (that obligation is stated explicitly in the docstring).

### 2. Contract

**Preconditions the code assumes (not enforced by types):**
- `network.nodes` is a dict of `str -> Node` where each node has at minimum: `.truth_value`, `.source`, `.source_hash`, `.metadata` (which may be `None`).
- `network.repos` (if used) maps repo name → string path; it will be coerced to `Path`.
- `node.source_hash`, if present, is either a 64-char full SHA-256 or a legacy 16-char truncation (prefix of the full hash).

**Postconditions:**
- Returns `(results, upgraded_count, sha_bumped_count)`.
- `node.source_hash`, `node.metadata["pinned_sha"]`, and `node.metadata["verified_at"]` may have been mutated in place (three specific branches).
- The network is **not** written to disk.

**Invariants:**
- Nodes with `truth_value != "IN"`, no `source`, or no `source_hash` are **never** touched.
- Iteration order is deterministic: `sorted(network.nodes.items())`.

### 3. Parameters

| Param | Type | Meaning |
|---|---|---|
| `network` | `Network` | The full network. Its `.nodes` are iterated; its `.repos` is used as a fallback for the `repos` parameter. |
| `repos` | `dict[str, Path] \| None` | Repo-name → local root path. If `None`, falls back to `{k: Path(v) for k, v in network.repos.items()}`. Passed to `resolve_source_path`. |
| `db_dir` | `Path \| None` | Directory alongside which expert repos live. Passed to `resolve_source_path` as a second lookup location. |
| `upgrade_hashes` | `bool` | If `True`, a 16-char `source_hash` that is a prefix of the current full hash is silently upgraded in place (counts toward `upgraded`). If `False`, the node is reported with `reason="truncated_hash"`. |
| `git_aware` | `bool` | If `True` and the node has `pinned_sha` metadata, git commands are used to short-circuit content hashing when the file (or its pinned line range) hasn't changed since the pinned commit. |

Edge cases:
- `repos=None` and `network.repos` empty → `repos` stays `None`; `resolve_source_path` handles `None` (falls back to `~/git/<repo>/...`).
- `db_dir=None` → skip the db_dir lookup.
- `upgrade_hashes=True` does not apply if no node is at a 16-char hash.

### 4. Return Value

Returns a 3-tuple:

1. **`results: list[dict]`** — each entry has the shape:
   ```python
   {
     "node_id": str,
     "old_hash": str,
     "new_hash": str | None,          # None == source_deleted
     "source": str,
     "source_path": str | None,       # None == source_deleted
     "reason": "source_deleted" | "truncated_hash" | "content_changed",
   }
   ```
   Note: the list is empty when no node is stale, when all upgrades were auto-applied, or when git-aware short-circuits fire.

2. **`upgraded: int`** — number of nodes whose truncated hash was replaced with a full hash in place. Only nonzero when `upgrade_hashes=True`.

3. **`sha_bumped: int`** — number of nodes whose `pinned_sha` was advanced to a newer commit. Only nonzero when `git_aware=True`.

**Caller obligations:** because of in-place mutation, the caller **must save the network** if `upgraded > 0`, `sha_bumped > 0`, or if it wants any updated `verified_at` values to persist (the docstring calls this out explicitly).

### 5. Algorithm

1. **Fall back to network repos.** If `repos is None` and `network.repos` exists, coerce to `Path` values.
2. **Initialize counters:** `results = []`, `upgraded = 0`, `sha_bumped = 0`.
3. **Iterate `sorted(network.nodes.items())`** — sorted by node ID string, deterministic.
4. **Skip quickly** — `truth_value != "IN"`, or no `source`, or no `source_hash` → `continue`.
5. **Resolve path.** Read `agent` from `node.metadata` (if any) and call `resolve_source_path(node.source, repos, db_dir, agent=agent)`.
   - If `None`: append `source_deleted` result and `continue`.
6. **Git-aware short-circuit (with pinned lines).** If `git_aware` and `pinned_sha` truthy:
   - If `pinned_lines` is present, parse `"<start>-<end>"`.
     - Malformed (`ValueError`/`IndexError`) → `pass` (fall through to content hashing).
     - Otherwise call `file_lines_changed_since(path, pinned_sha, ls, le)`:
       - **Unchanged lines** → call `get_file_commit_sha(path)`; if it differs from `pinned_sha`, update `metadata["pinned_sha"]`, `metadata["verified_at"]`, refresh `source_hash = hash_file(path)`, and increment `sha_bumped`. Then `continue` **regardless** of whether the SHA changed — the pinned range is considered fine.
       - **Changed lines** → fall through to content hashing.
   - Else (no `pinned_lines`):
     - `file_changed_since(path, pinned_sha)` returns `False` → `continue` (skip content hashing). No sha bump, because `git log -1 -- file` already yields the same SHA when nothing touched the file.
7. **Content hash.** `current_hash = hash_file(path)`.
8. **Compare hashes:**
   - **Different:**
     - If `len(node.source_hash) == 16` **and** `current_hash.startswith(node.source_hash)` → truncated-hash branch:
       - `upgrade_hashes=True` → mutate `node.source_hash = current_hash`, `upgraded++`, `continue` (node is *not* stale).
       - Otherwise → append `truncated_hash` result.
     - Else → append `content_changed` result.
   - **Same** (and `git_aware and pinned_sha`):
     - Fetch `new_sha = get_file_commit_sha(path)`. If different from `pinned_sha`, update `metadata["pinned_sha"]`, `metadata["verified_at"]`, `sha_bumped++`.
     - Note **no** `continue` here — but since we're at the end of the loop, it doesn't matter. This branch catches the case where the file content is unchanged but the last commit touching the file has moved (e.g., a commit that touched whitespace elsewhere, or a commit that didn't change content but is the current HEAD commit for that path).
9. Return `(results, upgraded, sha_bumped)`.

### 6. Side Effects

- **Disk reads:** `hash_file(path)` reads the entire file bytes and SHA-256s them. Called at most ~2× per node on the git-aware path (once by `hash_file`, once implicitly by nothing — `get_file_commit_sha` shells out, doesn't hash).
- **Subprocess calls (git):** possibly several per node when `git_aware=True` (`git log -1`, `git log ..HEAD`, `git diff --quiet HEAD`, `git diff -U0 <sha>`). Each has a 30 s timeout.
- **In-place mutation — three sites:**
  1. Truncated-hash upgrade (line `node.source_hash = current_hash`).
  2. Pinned-lines fast path (lines updating `pinned_sha`, `verified_at`, `source_hash` and incrementing `sha_bumped`).
  3. Content-unchanged/sha-bumped path (lines updating `pinned_sha`, `verified_at`, incrementing `sha_bumped`).
- **No persistence:** does not write `reasons.db`, does not call `network.save()`. Caller's responsibility.
- **No logging, no printing.**

### 7. Error Handling

Swallowed / conservative fallbacks:
- `file_lines_changed_since`, `file_changed_since` catch `subprocess.TimeoutExpired` **and** return `True` on nonzero returncode — i.e., on any git error, they force the content-hash fallback (no false "up to date" reports).
- `get_file_commit_sha` returns `None` on nonzero returncode (but see next line).
- Malformed `pinned_lines` format → caught inside `try/except (ValueError, IndexError) → pass`, so the code falls through to content hashing.

**Not caught:**
- **`get_file_commit_sha` has no `try/except TimeoutExpired`.** If git hangs on that call (called from the pinned-lines fast path and from the final `elif` branch), `subprocess.TimeoutExpired` will propagate out of `check_stale`. This is an asymmetry with the two `*_changed_since` helpers.
- **`hash_file` can raise `OSError`** (including `FileNotFoundError`) if the file disappears between `resolve_source_path`'s `.exists()` check and the `read_bytes()` call — a TOCTOU window. Not caught. The path is only known to exist at resolve time.
- Nothing validates `node.metadata` before assignment in the `sha_bumped` branches. This is safe *only because* those branches are gated on `pinned_sha` being truthy, which itself was read via `node.metadata.get(...) if node.metadata else None` — so `metadata` is guaranteed non-`None` by the time `node.metadata[...] = ...` runs.

### 8. Usage Patterns

- **Staleness report** (typical CLI): call with defaults, inspect `results`, do nothing with counts.
- **Migration mode:** call with `upgrade_hashes=True` to retire 16-char truncated hashes in one pass. Then the caller must save — otherwise next run reports `truncated_hash` again.
- **Performance mode:** call with `git_aware=True` on repos with many nodes and `pin_sources` already run. Pinned-lines nodes skip full-file hashing entirely when their line range is unchanged.
- **Combined mode:** `upgrade_hashes=True, git_aware=True` — both migrations and git optimization run in one pass.
- **After `hash_sources`**: `hash_sources` backfills missing hashes; `check_stale` will never flag nodes it can't hash (they're skipped for `not node.source_hash`). So order is normally `hash_sources` → `check_stale`.
- Tests `tests/test_check_stale.py`, `tests/test_check_stale_issue25.py`, `tests/test_pin_sources.py` exercise these paths. `test_pg_dispatch.py` suggests the command is also dispatched through a Postgres backend.

### 9. Dependencies

- **Stdlib:** `hashlib` (SHA-256), `subprocess` (git), `re` (hunk parsing in a sibling helper), `datetime.date`, `pathlib.Path`.
- **Internal:** `reasons.network.Network`.
- **Same-module helpers:**
  - `hash_file` — full SHA-256 of bytes.
  - `resolve_source_path` — agent-repo → db_dir → repos dict → `~/git/<repo>` fallback.
  - `get_file_commit_sha` — `git log -1 --format=%H -- <name>` in the file's parent dir.
  - `file_changed_since` — `git log <sha>..HEAD` + `git diff --quiet HEAD`.
  - `file_lines_changed_since` — `git diff -U0 <sha>`, then `parse_diff_hunks`.

### Assumptions Not Enforced by Types

- **The 16-char truncated-hash constant is a magic number.** Any legacy hash with a different length is treated as `content_changed`, not `truncated_hash`. Refactoring the truncation length would silently break upgrade detection.
- **`pinned_lines` format is `"<start>-<end>"` with 1-based line numbers.** The parser doesn't validate ordering or positivity here — that's enforced only on write in `pin_lines`. A hand-edited DB could produce invalid ranges that still parse successfully.
- **Node metadata is a mutable dict, not a pydantic/dataclass field.** Assignments to `node.metadata["pinned_sha"]` only work because metadata is a plain dict. If it were changed to a `frozen=True` dataclass this whole function would break.
- **`node.source_hash` is assumed to be a hex digest.** No validation; a corrupted value just compares unequal forever and gets reported as `content_changed`.
- **File identity is presumed stable between resolve and hash.** `resolve_source_path` returns `None` if the file doesn't exist *at that instant*; if it's deleted immediately after, `hash_file` raises.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — Contains the sibling helpers (`hash_sources`, `pin_sources`, `pin_update`, `pin_lines`) that produce the state this function reads back; understanding the write side clarifies what staleness really means.
- [function] `reasons/check_stale.py:resolve_source_path` — The lookup precedence (agent repo → db_dir → repos dict → `~/git/`) is critical to why some nodes incorrectly report `source_deleted`.
- [file] `tests/test_check_stale_issue25.py` — Documents a specific historical bug around staleness detection; will clarify edge cases not obvious from the source alone.
- [general] `git-aware-pinning` — The interaction of `pinned_sha` and `pinned_lines` and why the pinned-lines path bumps `source_hash` with a full-file hash even when only a range is relevant.
- [function] `reasons/check_stale.py:get_file_commit_sha` — The only git helper in this module that doesn't catch `TimeoutExpired`; worth understanding when the whole call can fail mid-flight.

## Beliefs

- `check-stale-mutates-network-in-place` — `check_stale` mutates `node.source_hash`, `node.metadata["pinned_sha"]`, and `node.metadata["verified_at"]`; the caller is responsible for persisting the network.
- `check-stale-skips-non-in-nodes` — Only nodes with `truth_value == "IN"` are considered; all other nodes are silently excluded from both `results` and the counts.
- `check-stale-returns-3-tuple` — Returns `(stale_results, upgraded_count, sha_bumped_count)`; `results` only lists nodes still considered stale or unresolvable, while upgrades are counted separately and removed from the results.
- `truncated-hash-is-16-hex-chars` — Legacy truncated hashes are assumed to be exactly 16 characters and a string prefix of the current 64-char SHA-256; any other mismatch reason is `content_changed`.
- `git-aware-pinned-lines-skips-content-hash` — When `git_aware=True` and a pinned line range is unchanged, `check_stale` skips `hash_file` entirely, but may still bump `pinned_sha` and refresh `source_hash` with the full-file hash if the file's latest commit SHA moved.
- `get-file-commit-sha-may-raise-timeout` — `get_file_commit_sha` does not catch `subprocess.TimeoutExpired`, unlike `file_changed_since`/`file_lines_changed_since`, so a git timeout on that call can propagate out of `check_stale` in git-aware mode.
