---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Both trees contain a copy of this file; verify the sync mechanism (copy script, symlink, or generated package) before editing.

The statement "both trees contain a copy of this file" refers specifically to the parallel `reasons/` and `reasons_lib/` directories at the repository root. Before any file in either directory is edited, you need to know whether the second tree is a symlink, a build-time copy, or a hand-maintained shim — because the answer changes which file is the edit target.

## What the codebase actually shows

### Two parallel package trees

`list_root_with_hidden_files` reports two package directories with nearly identical contents:

| Directory | Module count | Notes |
|---|---|---|
| `reasons/` | 34 modules | Includes `merkle.py` |
| `reasons_lib/` | 33 modules | **Missing `merkle.py`** |

Every other module (`api.py`, `ask.py`, `cli.py`, `network.py`, `storage.py`, `import_agent.py`, …) exists in both. Both directories are declared in `pyproject.toml:31`:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So both trees are *shipped packages*, not internal-only. That's why editing one without the other can silently break import-time behavior for anyone using `import reasons_lib`.

### What the documentation says

`CLAUDE.md:24` and `CLAUDE.md:35` claim the mechanism:

```
reasons/                # importable as `import reasons` (also aliased as reasons_lib for backwards compat)
...
reasons_lib/            # backwards-compat shim — re-exports from reasons/
```

The word **"shim"** is load-bearing. A shim is a hand-maintained re-export file (typically `from reasons.X import *`), not a copy and not a symlink. If that's accurate, every `reasons_lib/Foo.py` is a 1–3 line stub whose only purpose is to keep `import reasons_lib.Foo` working.

## Verifying the mechanism

### What was searched and what was found

The `search_sync_and_copy_mechanisms` search for `ln -s|symlink|copytree|shutil.copy|rsync|generated|sync` returned **123 matches, and none of them are a copy step between the two directories**. The matches are all:

- `.git/packed-refs:21` / `.git/FETCH_HEAD:21` — a *git remote branch* called `feat/sync-agent`. Unrelated.
- `.claude/skills/reasons/SKILL.md:154,209` — `reasons export-markdown` outputs a *generated snapshot*, not a package copy.
- `reasons/api.py:2016` `sync_agent(...)` and `reasons/cli.py:920` `cmd_sync_agent` — this `sync` refers to **synchronizing beliefs between agents** (`reasons/import_agent.py:17` shows `reasons sync-agent aap-expert ~/git/aap-expert/beliefs.md`). Completely unrelated to the directory duplication.
- `reasons/build_wiki.py:294,314,325,335` — variable name `generated_content` inside the wiki builder. Unrelated.
- `reasons/export_card.py:157` — a comment string `"Auto-generated card — review and customize…"`. Unrelated.

**Conclusion: no `copytree`, `rsync`, `shutil.copy`, Makefile target, or symlink-creation script between `reasons/` and `reasons_lib/` was found.** The `sync` keyword hits in this repo all refer to a completely different feature (belief synchronization between agents).

### Signals in the directory listing

The listing tool labels every `reasons_lib/*` entry as `"type": "file"` — not `"symlink"`. That's suggestive but not conclusive; many listing tools don't distinguish symlinks from files, and the observations don't include `ls -la` output.

More telling is the **missing `merkle.py`**. `reasons/merkle.py` exists (per the `reasons/` listing), but no `reasons_lib/merkle.py` appears in the `reasons_lib/` listing. A strict copy, a symlinked directory, or a generated package would all contain it. Its absence means one of:

1. `reasons_lib/` is a **hand-maintained re-export shim** (the `CLAUDE.md` claim) and `merkle.py` was added to `reasons/` without a corresponding shim being created — a drift bug that breaks `import reasons_lib.merkle`.
2. `reasons_lib/` is a **stale copy** that was last synced before `merkle.py` was added.

Both explanations rule out an automatic sync. There's no mechanism keeping the two trees in step.

### The `.github/workflows/tests.yml` gap

Only one workflow file exists (`.github/workflows/tests.yml`), but its contents were not read. This is the single best place to look for a hidden step like:

```yaml
- run: python -m pip install -e .
- run: python -c "import reasons_lib, reasons"   # would catch drift
```
or an explicit `diff -r reasons reasons_lib` check. Without reading this file, we cannot rule out that CI enforces parity (which would make `merkle.py` a build failure, not a published artifact).

## What to do before editing

Until you open a `reasons_lib/*.py` file, you cannot be certain. Run these checks:

1. **Read the shim itself.** `head -20 reasons_lib/api.py` and `head -20 reasons_lib/network.py`. If they're one or two lines starting with `from reasons.X import …`, they're re-export stubs; edit only `reasons/`. If they're hundreds of lines of real implementation, the two trees have diverged into a fork and edits must be reconciled in both.
2. **Check file modes.** `git ls-files -s reasons_lib/` — a mode of `120000` means symlink; `100644` means a real file. The observation `"type": "file"` for every entry suggests `100644`, but git introspection settles it.
3. **Check the CI workflow.** `.github/workflows/tests.yml` will reveal whether drift is caught (`import reasons_lib.merkle` fails a test) or silently shipped.
4. **Diff for drift.** `diff -r reasons reasons_lib` — if the two trees are supposed to be identical, this shows exactly where they're not. The `merkle.py` gap is already one confirmed divergence.

**Default assumption for a code change**: treat `reasons/` as the source of truth (per `CLAUDE.md:24`), make the change there, and — if the shim pattern holds — create or update the matching one-line file under `reasons_lib/`. Do not edit `reasons_lib/*.py` as if it were the primary implementation. If you discover any substantive logic in a `reasons_lib/*` file, stop and reconcile: the two trees are a fork, not a shim, and the "backwards-compat shim" comment in `CLAUDE.md` is stale.

## Topics to Explore

- [file] `reasons_lib/api.py` — The load-bearing file: whether it's a one-line `from reasons.api import *` or a full copy determines which tree is editable.
- [file] `.github/workflows/tests.yml` — The only CI definition; confirm whether it exercises `import reasons_lib` (which would catch drift like the missing `merkle.py`) or has an explicit sync check.
- [general] `git-ls-files-mode-bits` — `git ls-files -s reasons_lib/` distinguishes `100644` regular files from `120000` symlinks, ruling out or confirming the symlink hypothesis.
- [general] `reasons-lib-drift-audit` — `diff -r reasons reasons_lib` to quantify divergence; the missing `merkle.py` is already one confirmed gap.
- [file] `pyproject.toml` — Line 31 is the only place the two-package layout is declared; confirm there's no `package_data`/`exclude` that treats the trees differently.

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — CLAUDE.md:35 documents `reasons_lib/` as a re-export shim over `reasons/`, not a copy or symlink.
- `pyproject-ships-both-package-trees` — `pyproject.toml:31` declares `packages = ["reasons", "reasons_lib"]`, so both trees are importable in the installed distribution.
- `no-copy-tooling-connects-the-two-trees` — A search for `copytree|shutil.copy|rsync|ln -s|symlink` found no mechanism that mirrors `reasons/` into `reasons_lib/`.
- `reasons-lib-lacks-merkle-module` — `reasons/merkle.py` exists in the root listing, but no `reasons_lib/merkle.py` is present — evidence the trees have drifted.
- `sync-agent-is-belief-sync-not-code-sync` — `reasons/api.py:2016` (`sync_agent`) and `reasons/cli.py:920` (`cmd_sync_agent`) implement synchronization of *beliefs between agents*, unrelated to mirroring the two Python packages.
