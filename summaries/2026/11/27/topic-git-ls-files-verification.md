---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Run `git ls-files -s reasons_lib/` and `git ls-files -s reasons_lib` to check both contents and the directory entry itself.

## Explanation

### What `git ls-files -s` actually reports

`git ls-files -s` prints **one line per index entry** (not per working-tree file) in the form:

```
<mode> <object> <stage>\t<path>
```

The fields that matter for this diagnostic:

| mode | meaning |
|---|---|
| `100644` | regular file (what all 33 `.py` files should be) |
| `100755` | regular file, executable bit set |
| `120000` | symlink |
| `160000` | **gitlink** — submodule / nested repo pointer |

- `stage` is normally `0`; non-zero stages (`1`/`2`/`3`) mean an unresolved merge conflict.
- `git ls-files` only shows what is **tracked in the index**. A `.py` file sitting on disk but not added will not appear, even if it exists in `reasons_lib/`.

So the `-s` flag is doing double duty: asking "what is tracked?" **and** "in what form is it tracked?"

### Why run both variants — with and without the trailing slash

The trailing slash is a pathspec discriminator:

- **`git ls-files -s reasons_lib/`** — the slash says "match entries **under** `reasons_lib/`." This is the *contents* check. For a normal tracked directory you get one line per file. For a submodule you get **nothing**, because submodule contents live in the child repo and never appear in the parent index.

- **`git ls-files -s reasons_lib`** — no slash. On a normal tracked directory this typically also expands to files under it (git treats a bare directory as a prefix pathspec). But on a **gitlink** it returns exactly one line: `160000 <sha> 0 reasons_lib`. This is the *directory entry itself* check.

Put together, the two commands discriminate between four states:

| State of `reasons_lib` | `reasons_lib/` | `reasons_lib` |
|---|---|---|
| Normal tracked Python package | 33 × `100644` | 33 × `100644` (or empty on some git versions) |
| Submodule / accidental gitlink | (empty) | 1 × `160000` |
| Present on disk but untracked | (empty) | (empty) |
| Unresolved merge conflict | 33 lines with stages 1/2/3 | same |

### What we would expect for *this* repo

From the observations:

- `root_directory_entries` lists `reasons_lib/` as `type: "dir"` at the top level — it is a real working-tree directory, not something exotic.
- `reasons_lib_directory_entries` shows **33 `.py` files**, including the 3-line `reasons_lib/__init__.py` whose line 2 is `from reasons import *  # noqa: F401,F403` and whose docstring reads *"Backwards-compatibility shim — use ``import reasons`` instead."*
- `pyproject.toml` line 31 declares `packages = ["reasons", "reasons_lib"]` under `[tool.setuptools]`. Setuptools therefore treats `reasons_lib` as a first-class top-level package and will expect those 33 files to be present in the built sdist/wheel.
- `reasons_lib_references_in_python` reports **0 matches** for `reasons_lib` — nothing in the codebase imports it. That is entirely consistent with the shim: it exists only so old downstream callers that did `from reasons_lib import …` still work after the rename to `reasons`.

Given all that, a healthy result is:

- `git ls-files -s reasons_lib/` → **33 lines, all mode `100644`, stage `0`**
- `git ls-files -s reasons_lib` → **either the same 33 lines or empty** (git-version-dependent), **never a single `160000` line**

### What would be alarming (i.e. why this command pair is worth running)

- A `160000` line for `reasons_lib` → the package has been committed as a submodule. `pip install ftl-reasons` would then ship an **empty** `reasons_lib/` in the wheel, and `reasons_lib/__init__.py`'s compat shim would be unreachable at runtime — silent breakage for anyone still importing the old name.
- `reasons_lib/` returns nothing while the 33 files exist on disk → the package is untracked. An sdist built from git would not include them, so `pyproject.toml`'s package list would refer to ghosts.
- Non-zero `stage` values → `reasons_lib/__init__.py` (or neighbors) are in a merge conflict. The shim wouldn't resolve.
- Mode `100755` on a `.py` file → accidental exec bit, harmless but a smell.

### What's missing from the observations

The observations give us **working-tree** directory listings (`root_directory_entries`, `reasons_lib_directory_entries`) and `pyproject.toml`, but not the actual output of either `git ls-files` command. So everything above is a **prediction**, not a verified fact — we do not know the `.git/index` state. To close the loop, run both commands and inspect the mode column of every line.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — The 3-line compat shim; verify it is the file that must import cleanly from `reasons` for the legacy name to work
- [file] `pyproject.toml` — Confirm line 31's manual `packages = [...]` list and consider whether `find_packages()` would be safer than a hard-coded list
- [general] `gitlink-vs-directory` — Practice distinguishing mode `160000` from `100644` in `git ls-files -s` output so you can spot an accidental submodule in one glance
- [general] `sdist-tracking` — How `python -m build` / setuptools decide what ships; an untracked `reasons_lib/` is invisible to a git-based sdist even though it sits on disk
- [function] `reasons/__init__.py:Node` — The dataclasses (`Justification`, `Node`, `Nogood`) that `reasons_lib` re-exports via `from reasons import *`; these are the actual public symbols behind the shim

## Beliefs

- `reasons-lib-is-a-backwards-compatibility-shim` — `reasons_lib/__init__.py` line 2 runs `from reasons import *`, and its docstring says *"Backwards-compatibility shim — use ``import reasons`` instead."*
- `pyproject-lists-reasons-lib-as-a-package` — `pyproject.toml` declares `packages = ["reasons", "reasons_lib"]` under `[tool.setuptools]`, so setuptools treats both as top-level packages.
- `reasons-lib-contains-33-python-files` — The `reasons_lib/` directory has exactly 33 `.py` files (e.g. `api.py`, `cli.py`, `storage.py`), per `reasons_lib_directory_entries`.
- `reasons-lib-is-unreferenced-by-python-code` — Searching Python for the string `reasons_lib` returns 0 matches, consistent with it being a legacy alias rather than an active import.
- `reasons-lib-maps-to-reasons-via-star-import` — Because `reasons_lib/__init__.py` star-imports `reasons`, both packages expose the same public names (`Justification`, `Node`, `Nogood`), so downstream code can import either name interchangeably.
