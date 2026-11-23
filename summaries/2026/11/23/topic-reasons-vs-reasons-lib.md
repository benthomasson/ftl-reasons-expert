---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Resolve the duplicated package tree before trusting imports; determines where real logic lives.

# Resolving the Duplicated Package Tree (`reasons/` vs `reasons_lib/`)

## What the tree actually looks like

The repository ships **two top-level packages**, declared together in `pyproject.toml` line 31:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

That single line is the root of the "duplicated tree" concern — both names are installed and therefore both are importable. The question is which one holds real logic.

The evidence says: **`reasons/` is the source of truth; `reasons_lib/` is a thin re-export facade.**

### Evidence 1 — `reasons/__init__.py` defines real data types

`reasons/__init__.py` (lines 1–58) contains the only `@dataclass` definitions observed for the domain model:

- `Justification` (the SL/CP reason type with `antecedents`, `outlist`, `label`, `content_hash`)
- `Node` (the belief-network node with `truth_value`, `justifications`, `supporting_justification`, reverse-index `dependents`, etc.)
- `Nogood` (recorded contradiction across node IDs)

There is no counterpart definition anywhere in `reasons_lib/`.

### Evidence 2 — `reasons_lib/__init__.py` is a 2-line shim

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

(3 total lines, only 2 returned — the shim imports the entire `reasons` namespace.)

Every `reasons_lib/<module>.py` seen in the grep results follows the same pattern, e.g.:

- `reasons_lib/cli.py` line 2: `from reasons.cli import *`
- `reasons_lib/api.py` line 2: `from reasons.api import *`
- `reasons_lib/derive.py`, `compact.py`, `check_stale.py`, `import_agent.py`, `build_wiki.py`, `cluster.py`, `contradictions.py`, `export_card.py`, `export_markdown.py`, `hf.py`, `ask.py` — all identical shims.

No logic, only re-exports.

### Evidence 3 — the documentation says so explicitly

`CLAUDE.md`:

- line 24 — `reasons/` is "importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)"
- line 35 — `reasons_lib/` is a "backwards-compat shim — re-exports from `reasons/`"

`CHANGELOG.md` line 136 is the historical artifact: the Python API layer was *originally* called `reasons_lib.api`; that name is now a shim.

### Evidence 4 — real code imports from `reasons.*`, never `reasons_lib.*`

`grep_reasons_lib_references_all_files` returned **17 matches**, and **none are Python source imports**. The hits are:

- `CHANGELOG.md:136`, `CLAUDE.md:24` and `:35` (docs)
- `pyproject.toml:31` (packaging)
- `.forge/proposed-entries.json` lines 2–14 (generated wiki index)

Meanwhile `grep_primary_reasons_imports` shows **283 matches**, with concrete call sites like:

- `reasons/mcp_server.py:23` — `from reasons import api`
- `reasons/pg.py:943` — `from reasons.api import _STOP_WORDS`
- `reasons/pg.py:1324` — `from reasons.api import _TOPIC_STOP_WORDS`

And `imports_reasons_cli` shows `reasons/cli.py` using `from . import api` (a relative import inside `reasons/`), while `imports_reasons_lib_cli` shows `reasons_lib/cli.py` only doing `from reasons.cli import *`.

### Evidence 5 — entry points target `reasons`, not `reasons_lib`

`pyproject.toml` lines 27–29:

```toml
[project.scripts]
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

Even if `reasons_lib/` were deleted, both console scripts still resolve.

## How to "resolve" the tree before trusting imports

The practical procedure when you land in this repo and see two package names:

1. **Read `pyproject.toml`** — line 31 confirms both are installed, line 27–29 confirms the runtime entry points use `reasons`.
2. **Inspect `reasons_lib/__init__.py`** — it's 3 lines, and line 2 is a star import of `reasons`. Anything you import via `reasons_lib` *is* an object defined in `reasons`.
3. **Grep for `reasons_lib`** — 0 code references, 4 doc/packaging references. The shim exists for external downstream users, not for the current codebase.
4. **Grep for `from reasons|import reasons`** — 283 hits, with concrete internal imports in `reasons/mcp_server.py:23`, `reasons/pg.py:943`, `reasons/pg.py:1324`. That's where the logic originates.
5. **Snapshot the shim list** — every file under `reasons_lib/` that uses `from reasons.X import *` is a pure alias; any file under `reasons_lib/` that *doesn't* is either a stale copy or new logic smuggled into the wrong tree (a red flag).

## Consequences for trusting imports

- **Identity is preserved.** `from reasons_lib import Justification` and `from reasons import Justification` yield the *same class object*, because the shim executed `from reasons import *`. `isinstance` checks, dict keys, `abc` registrations, and equality all behave identically across the two namespaces.
- **Module identity is not preserved.** `reasons_lib.api.__name__` is `"reasons_lib.api"`, while `reasons.api.__name__` is `"reasons.api"`. Anything that keys on module path (pickling, dotted-string imports, tracing labels, `importlib` reload) sees two names for one implementation.
- **Monkey-patching is shared.** Mutating `reasons.api._STOP_WORDS` (referenced by `reasons/pg.py:943`) is visible through `reasons_lib.api` too, because both resolve to the same underlying module object.
- **The shim is not version-independent.** If `reasons/api.py` renames or removes a module that a shim still points to, `reasons_lib.api` breaks at import time even though the rest of the package works.

## What's missing from the observations

- The **full list of `reasons_lib/*.py` files** vs `reasons/*.py` — the grep only shows the shims that happened to match, so we can't be certain there isn't a non-shim file hiding in `reasons_lib/`.
- Whether **`reasons/__init__.py` defines `__all__`** — the file returned does not, which means `from reasons import *` in `reasons_lib/__init__.py:2` leaks `dataclass` and `field` from line 2 into `reasons_lib`'s namespace.
- Whether **any tests, scripts, or downstream users still `import reasons_lib`** — grep found none in the repo, but it may exist outside.
- Whether **`reasons_lib` has a `py.typed`, `__all__`, or deprecation warning** to signal the shim to callers.

## Topics to Explore

- [file] `reasons/api.py` — The real API layer that `reasons_lib/api.py` merely re-exports; verify what symbols are actually public.
- [function] `reasons/cli.py:main` — Confirms that `reasons` (not `reasons_lib`) is the runtime entry point wired through `[project.scripts]`.
- [file] `reasons_lib/__init__.py` — The 2-line shim that establishes the aliasing contract; check for any non-star-import logic before trusting it.
- [general] `import-star-export-semantics` — Why `from reasons import *` without `__all__` leaks `dataclass` and `field` into `reasons_lib`'s namespace.
- [repo] `.forge/proposed-entries.json` — The generated doc index that lists `reasons_lib-*` pages; confirms the shim tree has been documented but not re-implemented.

## Beliefs

- `reasons-is-shim-target` — `reasons_lib/__init__.py:2` does `from reasons import *`, so every symbol exposed by `reasons_lib` is defined in `reasons/`.
- `packaging-ships-both-trees` — `pyproject.toml:31` lists both `"reasons"` and `"reasons_lib"` in `[tool.setuptools] packages`.
- `entry-points-target-reasons` — `pyproject.toml:27-29` maps `reasons` and `reasons-mcp` console scripts to `reasons.cli:main` and `reasons.mcp_server:main`, never to a `reasons_lib.*` module.
- `first-party-code-avoids-shim` — Grep of `reasons_lib` returns 0 Python import sites; the only references are `CHANGELOG.md:136`, `CLAUDE.md:24`, `CLAUDE.md:35`, `pyproject.toml:31`, and `.forge/proposed-entries.json`.
- `reasons-has-no-all` — `reasons/__init__.py` (58 lines returned) contains no `__all__`, so the star-import shim in `reasons_lib/__init__.py` re-exports incidental names such as `dataclass` and `field` imported at `reasons/__init__.py:2`.
