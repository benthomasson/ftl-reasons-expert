---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Learn how Git stores `100644`, `100755`, and `120000`, and why `git ls-files -s` is authoritative for tracked symlink status.

# Git Mode Literals and `git ls-files -s` as the Source of Truth

## Important caveat about the observations

The five searches all returned **zero matches**:

- `ls-files` — 0
- `120000|100644|100755` — 0
- `is_symlink|islink|readlink|symlink|lstat` — 0
- `subprocess|git …` — 0
- `tracked|untracked|st_mode|S_ISLNK|S_ISREG|stat(` — 0

That means **the codebase as searched contains no files, functions, or line numbers to cite** for this topic. There is no index parser, no `git ls-files` call, no symlink inspection, and no tracked-file logic anywhere the patterns could reach. So I can explain the *Git contract* precisely, but I cannot ground it in this repository. To connect the explanation to code, we would need the Git integration layer (likely a subprocess wrapper or a native index reader) that the searches failed to surface — possibly because it lives in a dependency, a submodule, a generated file, or under a differently named symbol (e.g. `porcelain`, `index_reader`, `GitRepo`).

The explanation below is therefore the **authoritative Git behavior** that any implementation of this feature must rely on, independent of what was or wasn't found in the search.

---

## 1. What the three literals are

Git stores a **32-bit mode** per index entry, but only a small, normalized subset is ever written:

| Literal  | Meaning                                      | Object type |
|----------|----------------------------------------------|-------------|
| `100644` | Regular file, **not** executable             | blob        |
| `100755` | Regular file, executable                     | blob        |
| `120000` | Symbolic link                                | blob        |
| `040000` | Directory (tree objects, not index entries)  | tree        |
| `160000` | Gitlink / submodule                          | commit      |

These literals are **not Unix permission bits**. `100755` does not mean `chmod 755`; it means "the single executable bit is set." Git does not track group/other permission bits or setuid — it normalizes on `git add`. That is why `core.fileMode` exists: on filesystems that cannot represent or do not report the exec bit (FAT, NTFS, some CIFS mounts), Git must be told to ignore it, otherwise every add looks like a `100644 ↔ 100755` flip.

## 2. How a symlink is stored

- On `git add`, Git calls `lstat` on the path. If `S_ISLNK(st_mode)` is true, it **reads the link target** (via `readlink`) and writes that target string as an ordinary **blob**.
- The index entry is recorded with mode `120000` and the blob's object id. Nothing about "this is a symlink" lives in the blob itself — the *mode* is the only flag.
- On checkout, Git sees `120000` and calls `symlink(blob_contents, path)` instead of writing a regular file. If the platform cannot create symlinks, `core.symlinks=false` makes Git write the target path as a **regular text file** instead — so a `120000` index entry can be materialized as a plain file on disk.
- On a case-insensitive / symlink-less checkout, the working tree can therefore disagree with the index in *type*, not just content. This is the whole reason working-tree `lstat`/`islink` is not a reliable signal.

## 3. Why `git ls-files -s` is authoritative

`git ls-files -s` (equivalently `--stage`) dumps the **index verbatim**:

```
<mode> <object-id> <stage>\t<path>
```

Properties that make it the ground truth for *tracked* status:

1. **It reads the index, not the working tree.** There is no `lstat` against disk, no `.gitignore` heuristic, no rename detection, no merge resolution. Each line is exactly one index entry.
2. **It reports the mode Git actually recorded**, so you can classify tracked symlinks with a single comparison: `mode == "120000"`. No heuristics.
3. **It reports stage bits.** `1` is normal; `2`/`3` are merge conflicts. `git ls-tree HEAD` does not see staged-but-uncommitted changes; `git ls-files -s` does.
4. **It reports what Git considers "tracked"**, by definition: an entry in the index is tracked; an absent path is untracked regardless of what's on disk.

Contrast with common alternatives that are *not* authoritative for the tracking question:

- **`os.path.islink(path)` / `readlink` / `lstat`** — describes the **working copy only**. It says nothing about what Git tracks. A symlink on disk can be untracked, `.gitignore`'d, staged as a regular file (rare but possible with `core.symlinks=false` round-trips), or an unmerged typechange.
- **`git status --porcelain`** — compares *worktree + index + HEAD* and applies rename/ignore logic. It reports *differences*, not the canonical mode. Fine for change detection, wrong for "is this tracked as a symlink."
- **`git ls-tree <rev>`** — the same mode vocabulary, but reflects a **tree object at that revision**, not the index. Missing staged changes.
- **`git cat-file -p <blob>`** — gives you the link target text, but not the fact that it *is* a link. Only the mode (from the index or a tree) carries that.

The correct algorithm for "is this path tracked as a symlink?" is therefore:

```
git ls-files -s -- <path>   # parse "<mode> ...\t<path>"
mode == 120000 ? tracked symlink : not
```

`--debug` can be appended to see cached stat info (useful to detect unmerged/assume-unchanged/skip-worktree states), but `-s` alone is the canonical view.

---

## Topics to Explore

- [general] `git-index-format` — Read `Documentation/gitformat-index.adoc` in the Git repo to see how the 32-bit mode is actually stored in `.git/index`, and why the mode is limited to 16 bits even though it is written as 32.
- [general] `core.fileMode-and-core.symlinks` — These two config flags directly determine whether `100755` and `120000` survive a checkout; understanding them explains most cross-platform symlink bugs.
- [general] `git-ls-files-variants` — Compare `-s`, `-t`, `--debug`, `--eol`, and `--format` to see which output is stable to parse and which is human-oriented.
- [general] `symlink-to-blob-model` — Confirm that `git add` of a symlink stores `readlink` output as a blob via `S_ISLNK` in `read-cache.c` / `entry.c`, so that the "target is file content" mental model is exact.
- [general] `gitportable-vs-porcelain` — Distinguish plumbing commands (`ls-files`, `ls-tree`, `cat-file`) from porcelain (`status`, `diff`); only plumbing is safe to parse for authoritative decisions.

## Beliefs

- `git-ls-files-s-reads-index-not-worktree` — `git ls-files -s` emits one line per index entry and does not `lstat` the working tree, so it reflects tracked state even when checkout and disk disagree.
- `mode-literal-120000-identifies-tracked-symlink` — A mode of `120000` in `git ls-files -s` output is the definitive marker of a symlink tracked by Git; the blob content alone does not distinguish a symlink from a text file.
- `symlink-target-is-stored-as-blob-content` — Git stores a symlink's target path as the blob object referenced by the `120000` index entry; the mode bit, not the blob, is the type tag.
- `git-mode-is-not-unix-permissions` — `100755` encodes only the executable bit; group/other permission bits and setuid are never tracked, and `core.fileMode` exists to suppress spurious exec-bit changes on filesystems that lack it.
- `codebase-lacks-git-mode-handling` — The provided observations contain no matches for `ls-files`, mode literals, symlink checks, subprocess Git calls, or tracked-file logic, so no file or function in this repository currently implements or consumes the `100644`/`100755`/`120000` contract.
