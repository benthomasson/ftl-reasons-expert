---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: the repo ships two packages with near-identical files; investigate how they stay in sync (or don't).

# How `reasons` and `reasons_lib` stay in sync

## Verdict up front

Based on the observations, the two packages are **not** kept in sync by copying files. The shipped `reasons_lib` package is a **backwards-compatibility shim** that re-exports the real code in `reasons/`. At least for the one file shown (`reasons_lib/cli.py`), there is no duplicate source — it's a one-line wildcard import. The premise of "near-identical files" is only true in the sense that `reasons_lib` likely contains a small shim module per public module; it is **not** true for `cli.py`, which contains no duplicated logic.

## Evidence

### 1. Packaging ships both names

`pyproject.toml:31` declares both packages so that `import reasons` and `import reasons_lib` both resolve after install:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

The project name and entry point are unaffected — `reasons = "reasons.cli:main"` still points at the canonical package; `reasons_lib` is purely an import alias for library consumers.

### 2. The doc-level contract says "shim", not "fork"

`CLAUDE.md:24`:
> `reasons/` # importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)

`CLAUDE.md:35`:
> `reasons_lib/` # backwards-compat shim — re-exports from reasons/

So the intended design is delegation, not duplication.

### 3. The observed `reasons_lib` file is a re-export, not a copy

`imports_reasons_lib_cli` shows the **entire** `reasons_lib/cli.py` contains only:

```python
from reasons.cli import *
```

That's the sync mechanism in a nutshell: there is nothing to keep in sync, because `reasons_lib.cli` *is* `reasons.cli`'s namespace. Any change to `reasons/cli.py` is automatically reflected the moment you `import reasons_lib.cli`.

Compare with the real file, `reasons/cli.py`, which imports `argparse`, `json`, `os`, `sys`, `pathlib.Path`, `importlib.metadata.version`, and — importantly — uses a **relative** import (`from . import api`). Because the relative import resolves against the `reasons` package regardless of which top-level name the caller used, the shim does not need any path fix-ups.

### 4. No file-sync tooling exists

`sync_or_copy_tooling_search` searched for `sync|rsync|shutil.copy|copyfile|filecmp|mirror|duplicate|identical`. The hits are all noise relative to package duplication:

- `.git/hooks/commit-msg.sample:18` — Git sample, unrelated.
- `.git/packed-refs` and `.git/FETCH_HEAD` — branches named `feat/sync-agent` and `fix/deduplicate-*`, unrelated to file mirroring.
- `CHANGELOG.md:28` — `sync-agent` **command** for belief reconciliation, not packaging.
- `CHANGELOG.md:64, 70` and `reasons/api.py:1240+`, `reasons/api.py:4976+` — the deduplication feature of the *belief system*, not source files.
- `reasons/api.py:2016+` — `sync_agent()` for post-import belief updates.
- `reasons/build_wiki.py:60-63` — rendering `duplicate_of` belief metadata.

There is **no** `Makefile` target, no `scripts/sync_packages.py`, no `shutil.copyfile` loop, and no CI job that regenerates one package from the other. `.github/workflows/tests.yml` is the only workflow, and the observations don't show it doing any parity enforcement.

### 5. Directory conventions confirm the shim pattern

The `CLAUDE.md` project tree places `reasons_lib/` on a single "shim" line (line 35), parallel to `tests/`. It does not enumerate per-module files under `reasons_lib/`, which is consistent with the shim containing one small file per public module (e.g. `reasons_lib/api.py` → `from reasons.api import *`, `reasons_lib/__init__.py` → `from reasons import *`). That is how `import reasons_lib.api` would continue to resolve for legacy callers.

## What this means in practice (and the risk)

- **Scenarios that are safe by construction:** any change inside `reasons/cli.py` (or any other re-exported module) is instantly visible through `reasons_lib`. There is no drift window, no generation step, no CI check needed.
- **Scenarios that *would* drift (not observed):** if any `reasons_lib/*.py` ever contained a literal copy instead of a wildcard import, nothing in the repo would catch it — no sync tooling, no mirror check, no test asserting parity. The `tests.yml` workflow is the only guard, and the observations don't show it importing `reasons_lib`.
- **Namespace subtlety:** `from reasons.cli import *` only re-exports names that are public (no leading underscore) or listed in `reasons.cli.__all__`. If `reasons/cli.py` ever defines `__all__` incompletely, the `reasons_lib` shim would silently drop names — a real way the two namespaces could diverge without any build error.

## What's missing from the observations

To fully confirm the story, we'd need:

1. **A listing of `reasons_lib/`.** The observations show only `reasons_lib/cli.py`. If there are per-module files (`api.py`, `storage.py`, `network.py`, …) that are also one-line `from reasons.X import *`, the "shim" story is airtight. If some are literal copies, the premise "near-identical files" is real and there is *no* sync mechanism for those.
2. **`reasons_lib/__init__.py`.** Whether `import reasons_lib` (top-level) works depends on it doing `from reasons import *` (or equivalent).
3. **`reasons/__init__.py` and its `__all__`.** Determines what the top-level shim actually exports.
4. **The body of `.github/workflows/tests.yml`.** Confirms whether CI ever imports `reasons_lib` (i.e., whether parity is even smoke-tested).
5. **Test files importing `reasons_lib`.** The `reasons_lib` reference search returned zero hits in `tests/`, suggesting legacy imports are *not* covered by the 211-test suite.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Verify that the top-level shim wildcard-imports `reasons`, which is what makes `import reasons_lib` behave like `import reasons`.
- [file] `reasons_lib/` — Enumerate the directory to distinguish true one-line shims (`from reasons.X import *`) from any literal copy-pasted modules that would actually drift.
- [file] `reasons/__init__.py` — Check for an `__all__`; if present and incomplete, `from reasons import *` in the shim would silently drop exports and let the two packages diverge.
- [file] `.github/workflows/tests.yml` — Confirm whether CI imports `reasons_lib` at all, i.e., whether backward-compat imports are regression-tested.
- [repo] `benthomasson/ftl-reasons` — Inspect git history for when `reasons_lib` was introduced (renamed from `rms` per `CLAUDE.md` line 5), to see whether it was ever a literal fork before being collapsed into a shim.

## Beliefs

- `reasons-lib-is-a-reexport-shim` — `reasons_lib/cli.py` consists solely of `from reasons.cli import *`, so it contains no duplicated CLI logic to keep in sync.
- `pyproject-ships-both-packages` — `pyproject.toml:31` lists both `"reasons"` and `"reasons_lib"` under `[tool.setuptools] packages`, so both are installed from one distribution.
- `reasons-lib-has-no-file-sync-tooling` — The codebase contains no `rsync`/`shutil.copyfile`/`filecmp`/mirror script targeting `reasons_lib`; all apparent "sync"/"duplicate" matches belong to the belief-deduplication feature or git metadata.
- `cli-uses-relative-api-import` — `reasons/cli.py` imports its API via `from . import api`, which is why the same module works unchanged behind the `reasons_lib` shim.
- `reasons-lib-is-a-backwards-compat-alias` — `CLAUDE.md:24` and `CLAUDE.md:35` explicitly document `reasons_lib` as an alias/shim for the renamed `reasons` package, not a maintained fork.
