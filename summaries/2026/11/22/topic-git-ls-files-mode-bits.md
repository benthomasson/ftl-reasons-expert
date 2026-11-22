---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: `git ls-files -s reasons_lib/` distinguishes `100644` regular files from `120000` symlinks, ruling out or confirming the symlink hypothesis.

## What `git ls-files -s reasons_lib/` actually tells you

`git ls-files -s` queries Git’s **index** and prints the staged mode, object hash, stage number, and path for tracked files:

```text
<mode> <object> <stage>\t<path>
```

The mode is the discriminator:

- `100644` — regular non-executable file
- `100755` — regular executable file
- `120000` — symbolic link
- `160000` — gitlink / submodule

So output like this would mean “regular file”:

```text
100644 blob abc123... 0	reasons_lib/__init__.py
```

And output like this would mean “symlink”:

```text
120000 blob def456... 0	reasons_lib/some_alias.py
```

For a symlink, the blob contents are the link target, so `git cat-file -p <hash>` would show where the symlink points.

One subtlety: `git ls-files -s reasons_lib/` with the trailing slash queries tracked paths *under* `reasons_lib/`. If `reasons_lib` itself were a symlink, the entry for the symlink itself would normally be checked with `git ls-files -s reasons_lib` without the slash. That would show something like:

```text
120000 blob abc123... 0	reasons_lib
```

So the stronger verification is usually to run both:

```bash
git ls-files -s reasons_lib/
git ls-files -s reasons_lib
```

Also note this reports Git’s index, not arbitrary filesystem state. If someone locally replaced a tracked file with a symlink but did not stage it, `git ls-files -s` would still show the index mode. `git status --short` or `git diff --summary` would catch that kind of worktree typechange.

## What the supplied observations show

The observations are **consistent with** `reasons_lib` being a real package directory:

- `list_top_level_directory` shows `reasons_lib/` with `"type": "dir"`.
- `list_reasons_lib_directory` reports 33 entries under `reasons_lib/`, all typed `"file"`, including `reasons_lib/__init__.py`, `reasons_lib/api.py`, `reasons_lib/storage.py`, etc.
- `pyproject.toml` line 31 lists `reasons_lib` in `[tool.setuptools].packages`:
  ```toml
  packages = ["reasons", "reasons_lib"]
  ```

However, the observations **do not include the actual output of `git ls-files -s reasons_lib/`**. The searches for `symlink|120000|100644|ls-files` found zero matches, which only means those terms do not appear in the searched text; it does not establish Git modes.

So the correct conclusion from the supplied observations is:

- The command **would** distinguish `100644` regular files from `120000` symlinks.
- If `git ls-files -s reasons_lib/` shows only `100644` / `100755` entries and no `120000`, the symlink hypothesis is **ruled out** for tracked contents.
- If any entry is `120000`, the symlink hypothesis is **confirmed** for that specific path.
- If `git ls-files -s reasons_lib/` returns nothing and `git ls-files -s reasons_lib` returns `120000`, then `reasons_lib` itself is likely a symlink.
- The provided observations alone are **insufficient** to make the final call; the actual Git mode output is missing.

Also, `.forge/proposed-entries.json` contains diary-entry paths like `entries/2026/04/23/reasons_lib-api.md`. Those are generated documentation entries about `reasons_lib` modules, not the tracked package files themselves, so they do not establish whether `reasons_lib/` contains symlinks.

## Topics to Explore

- [general] `git-index-modes` — Learn how Git stores `100644`, `100755`, and `120000`, and why `git ls-files -s` is authoritative for tracked symlink status.
- [file] `pyproject.toml` — The `[tool.setuptools]` section lists `reasons_lib` as a package, so packaging behavior would be affected if it were a symlink.
- [file] `reasons_lib/__init__.py` — The package marker; checking its Git mode directly distinguishes a regular file from a symlink in the index.
- [general] `git-ls-files-verification` — Run `git ls-files -s reasons_lib/` and `git ls-files -s reasons_lib` to check both contents and the directory entry itself.
- [file] `.forge/proposed-entries.json` — Shows generated diary entries for `reasons_lib` modules, but not Git tracking modes.

## Beliefs

- `reasons-lib-is-a-setuptools-package` — `pyproject.toml` line 31 includes `reasons_lib` in `[tool.setuptools].packages`.
- `reasons-lib-workspace-listing-has-33-file-entries` — The workspace listing reports 33 entries under `reasons_lib/`, all typed as files, from `__init__.py` through `verify.py`.
- `git-mode-120000-identifies-symlinks` — In `git ls-files -s` output, `120000` identifies a symlink while `100644` identifies a regular non-executable file.
- `observed-data-lacks-git-ls-files-output` — No actual `git ls-files -s reasons_lib/` output is present in the observations, so the symlink conclusion remains to be verified.
