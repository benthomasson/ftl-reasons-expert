---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: resolve_source_path in reasons/check_stale.py

# `resolve_source_path` — Explanation

## Purpose

`resolve_source_path` turns a **logical source string** (e.g. `"repo-name/path/to/file.md"`) that a belief node stores into an **actual filesystem path**, but only if that file exists. It exists because nodes remember their *origin* in a human-friendly string form (repo name + relative path, or a bare relative path), while staleness checking needs to open the actual file to hash it. The function bridges human-readable provenance and filesystem reality, returning `None` when the source can no longer be found (`check_stale` then reports `source_deleted`).

The docstring explicitly frames a fallback chain: **agent repo → db_dir → repos dict by first component → `~/git/`**.

## Contract

- **Precondition:** `source` is a string; the caller has (or hasn't) populated `repos`, `db_dir`, and `agent`. Nothing else is required.
- **Postcondition:** Returns a `Path` that exists on disk, or `None`. Never returns a path it hasn't verified with `.exists()`.
- **Invariant:** The returned path is *only* produced after an `.exists()` check; resolution is not pure string concatenation.
- **Not enforced by types:** the code assumes `source` is *relative* (not absolute). If `source` were absolute (e.g. `/etc/passwd`), the `repos[agent] / source` and `repos[repo_name] / rel_path` joins would treat it as override and ignore the parent (Path semantics: `Path("a") / "/b"` → `Path("/b")`). The type system does not prevent this.

## Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `source` | `str` | Origin string. Either `"repo/relative/path"` or a bare `"relative/path"`. | Empty/falsy → immediate `None`. A path with no `/` is treated as CWD-relative. |
| `repos` | `dict[str, Path] \| None` | Map of repo name → root path. Populated by callers from `network.repos` (converted to `Path`). | `None` or missing key just skips those branches. |
| `db_dir` | `Path \| None` | Directory holding `reasons.db`; expert repos keep sources alongside it. | If `None`, the branch is skipped. |
| `agent` | `str \| None` | Agent name for agent-imported beliefs. | Only used if `agent in repos`. |

## Return Value

- `Path` — an existing file. Caller may hash it.
- `None` — source could not be resolved on disk. **Callers must handle `None`**:
  - `check_stale` emits a `"reason": "source_deleted"` result.
  - `hash_sources`, `pin_sources`, `pin_update`, `pin_lines` silently skip the node.

## Algorithm

1. **Empty guard.** `if not source: return None`.
2. **Agent repo attempt.** If `agent` is set, `repos` is set, and `agent in repos`, try `repos[agent] / source`. If the resulting file exists, return it. *This handles sources that are relative to the importing agent's repo, not a repo keyed by the first path segment.*
3. **db_dir attempt.** If `db_dir` is set, try `db_dir / source`. If it exists, return it. *This covers expert repos whose sources live next to `reasons.db`.*
4. **Split once.** `parts = source.split("/", 1)`. Note `maxsplit=1` — this is crucial: it means `"repo/sub/dir/file.md"` → `("repo", "sub/dir/file.md")`, keeping the remainder intact.
5. **No slash case.** If `len(parts) < 2`, treat the whole source as a bare relative path: `p = Path(source)`; return it if it exists else `None`. (Relative to the process CWD.)
6. **Repo-name case.** Unpack `repo_name, rel_path`.
   - If `repos` is present and `repo_name in repos`: `p = repos[repo_name] / rel_path`.
   - Else: fall back to `Path.home() / "git" / repo_name / rel_path`.
7. **Existence gate.** Return `p` if `p.exists()` else `None`.

Note the ordering: agent and db_dir are tried *before* the split-based lookup, so those take precedence even when the first segment of `source` happens to match a repo name. This means two different repos could both be plausible candidates and the earlier branch wins.

## Side Effects

- **Filesystem reads only:** repeated `Path.exists()` calls (stat on the filesystem). No writes, no mutations of arguments.
- **`Path.home()`** may internally consult environment (`HOME` / user database); rarely raises `RuntimeError` if no home is determinable — not handled here.
- No subprocess, no hashing, no network.

## Error Handling

- **No explicit exceptions.** The function is designed to fail closed by returning `None`.
- `.exists()` swallows most OS errors by returning `False` (e.g. permission denied, broken symlink loop), so a source that exists but is unreadable will appear "deleted."
- `Path.home()` raising `RuntimeError` is the only realistic uncaught exception, and only in environments without a resolvable home directory.

## Usage Patterns

All four callers in the module (`check_stale`, `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`) follow the same pattern:

```python
agent = node.metadata.get("agent") if node.metadata else None
path = resolve_source_path(node.source, repos, db_dir, agent=agent)
if path is None:
    # node's source is gone — skip or flag
```

**Caller obligations:**
- Build `repos` from `network.repos` when not supplied (the callers do this).
- Treat `None` as "missing source," never assume resolution succeeds.
- Pass the node's `metadata["agent"]` so agent-relative lookups work.

## Dependencies

- **Standard library only:** `pathlib.Path`.
- No imports from the rest of the module's dependencies (`hashlib`, `subprocess`, `re`) are needed by this function — it is pure path resolution.
- Relies on the caller-side convention that `network.repos` maps repo *names* to filesystem roots and that node sources are relative to those roots (or to `db_dir` / the agent's repo).

## Assumptions Not Enforced by Types

1. `source` is relative; absolute sources would bypass the repo-root join.
2. `repos` keys are exactly the first segment of `source` (or the agent name).
3. `db_dir` and repo roots are directories whose children are the files named by `source`.
4. The `~/git/<repo>/<path>` fallback assumes the conventional checkout layout.
5. File existence is the only test for correctness — a wrong-but-existing file with a matching path will be returned and then hashed, potentially producing false "fresh" or "stale" verdicts.

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — primary consumer; shows how `None` maps to `source_deleted` and how `git_aware` interacts with resolution
- [file] `reasons/network.py` — defines `Network.repos`, which feeds the `repos` parameter
- [function] `reasons/check_stale.py:hash_sources` — same resolver used for backfilling missing hashes
- [file] `tests/test_check_stale_issue25.py` — likely exercises the agent-repo / db_dir precedence and fallback paths
- [general] `source-path-conventions` — how `source` strings are generated at import time (agent vs. expert vs. bare path)

## Beliefs

- `resolve-source-path-requires-file-existence` — The function returns a `Path` only after verifying `path.exists()`; otherwise it returns `None`.
- `resolve-source-path-checks-agent-repo-first` — When `agent` is present in `repos`, that repo is consulted before `db_dir` and the split-based lookups.
- `resolve-source-path-splits-once` — `source.split("/", 1)` preserves remaining path segments, so multi-level relative paths survive repo-name extraction.
- `resolve-source-path-falls-back-to-git-home` — If the first segment isn't a key in `repos`, resolution uses `~/git/<repo_name>/<rel_path>`.
- `resolve-source-path-returns-none-on-missing-source` — Empty `source` and any unresolved/ nonexistent path yield `None`, which callers interpret as `source_deleted`.
