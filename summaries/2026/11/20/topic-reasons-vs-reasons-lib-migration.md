---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The dual-tree layout and why canonical code still routes through legacy shims; the answer determines whether `reasons_lib/` can ever be deleted.

## The dual-tree layout in this repo

Two top-level package directories ship from the same distribution:

| Path | Role | Evidence |
|---|---|---|
| `reasons/` | Canonical package | Defines `Justification`, `Node`, `Nogood` in `reasons/__init__.py:1-59`; holds the 5,886-line functional API in `reasons/api.py` |
| `reasons_lib/` | Two-file re-export shim | `reasons_lib/__init__.py:3` → `from reasons import *`; `reasons_lib/api.py:3` → `from reasons.api import *` |

Both are declared for installation in `pyproject.toml:31`:
```toml
packages = ["reasons", "reasons_lib"]
```

The two console scripts in `pyproject.toml:27-28` point only at the canonical module:
```toml
reasons      = "reasons.cli:main"
reasons-mcp  = "reasons.mcp_server:main"
```

So the "dual tree" is: one real package, plus a thin alias that preserves the *old* distribution's import paths.

## The framing is inverted: canonical code does **not** route through the shims

The question assumes canonical code still routes through legacy shims. The observations say otherwise:

- **`reasons/api.py` imports only from the canonical tree.** Look at `reasons/api.py:20-24`:
  ```python
  from . import Justification
  from . import pubsub
  from .metadata import build_meta
  from .network import Network
  from .storage import PROPOSALS_SCHEMA, Storage
  ```
  Every one of these is a relative import inside `reasons/`. Nothing in `reasons/` imports from `reasons_lib`.

- **`reasons/api.py` also calls only canonical helpers in its body** — `_pg_dispatch` imports `from .pg import PgApi` (line ~200 of the file), `_with_network` uses `Storage` from `.storage`, etc.

- **Repo-wide search for `reasons_lib` returns 0 matches** (`reasons_lib_references` observation). No source file, no test, no config references `reasons_lib` by name.

- **`reasons_lib` is a leaf, not a dependency.** The only edges are `reasons_lib/__init__.py:3` → `reasons` and `reasons_lib/api.py:3` → `reasons.api`. The arrow points *out of* the shim, never *into* it.

The dependency graph is one-directional:

```
reasons_lib/__init__.py ──→ reasons/__init__.py
reasons_lib/api.py      ──→ reasons/api.py
```

There is no cycle, and no internal consumer. The `test_package_imports` observation returned 0 matches as well — arguably suspicious (see *What's missing* below), but it reinforces that nothing in the current tree depends on the shim.

## What the shim is actually for

The only surviving mentions of `reasons_lib` are documentation:

- `CHANGELOG.md:136` — "Python API layer (`reasons_lib.api`) returning dicts…" (historical name)
- `CLAUDE.md:24` — "reasons/ … (also aliased as reasons_lib for backwards compat)"
- `CLAUDE.md:35` — "reasons_lib/ … backwards-compat shim — re-exports from reasons/"

So `reasons_lib/` exists **only** to keep pre-rename external import sites working. It is a public API compatibility surface, not an internal module. The two shim files cover exactly two import paths a legacy user could have written:

- `import reasons_lib` (or `from reasons_lib import Justification/Node/Nogood`)
- `from reasons_lib.api import …`

Notably, the shim is **incomplete**: `reasons_lib/` contains only `__init__.py` and `api.py`. There is no `reasons_lib/network.py`, `reasons_lib/storage.py`, `reasons_lib/cli.py`, etc. So a legacy caller who did `from reasons_lib.network import Network` is *already broken today*. Any external consumer still alive can only be using the two covered paths.

## Can `reasons_lib/` ever be deleted?

**Yes, from this repo's perspective — nothing internal breaks.** The deletion is safe against every artifact in the observations:

1. `pyproject.toml:31` would just lose `"reasons_lib"` from the list.
2. Console scripts already target `reasons.*`.
3. No source file imports it (`reasons_lib_references`: 0 matches).
4. Tests don't import it (`test_package_imports`: 0 matches).
5. `reasons/api.py` has zero dependency on the shim.

The only thing keeping `reasons_lib/` alive is the **out-of-tree + already-partially-broken legacy contract**: anyone on the old distribution who did `import reasons_lib` or `from reasons_lib.api import …`. That is a release-management decision, not a code dependency. A safe deletion path looks like:

1. Add a `DeprecationWarning` on import of `reasons_lib` (not currently present in the observations).
2. Ship one or two minor versions with the warning.
3. Yell in `CHANGELOG.md` next to the line-136 historical note.
4. Bump a major, delete `reasons_lib/`, drop the entry in `pyproject.toml:31`.
5. Optionally keep a tiny `reasons_lib` PyPI stub package forever if you want absolute safety for downstream imports.

The **only** question that can't be answered from the observations is whether *any external package on PyPI or in this org still imports `reasons_lib`*. That's the real gate — not anything in this repo.

## What's missing from the observations

To make the deletion decision with more confidence, we'd want:

- **Git history** of when `reasons_lib` → `reasons` happened, and whether a `DeprecationWarning` was ever scheduled.
- **Downstream dependency graph** — who depends on the `ftl-reasons` distribution and whether those consumers import `reasons_lib`.
- **`__all__` declarations** in `reasons/__init__.py` and `reasons/api.py` — `from reasons import *` only re-exports the public names it decides on. Without `__all__`, wildcard exports include every non-underscore top-level name, which is broad but fragile.
- **A re-run of the test-import scan** — `test_package_imports` returning 0 is unusual; the regex `from reasons import|import reasons` would *not* match the very common `from reasons.api import foo`, which may explain the zero.
- **`reasons_lib` file count** — observations only show 2 files; confirm via `find reasons_lib -type f`.
- **Any editable-install or dev-tool config** (e.g. `tox.ini`, `.github/workflows`) that might reference the shim in a way `pyproject.toml` alone doesn't reveal.

## Topics to Explore

- [file] `reasons/api.py:1-60` — The canonical API surface that `reasons_lib/api.py` re-exports by `*`; confirms what the shim actually exposes.
- [file] `pyproject.toml` — Verify `packages = ["reasons", "reasons_lib"]` on line 31 and the entry points on lines 27-28; these are the only build-time references to the shim.
- [general] `downstream-consumers-of-reasons_lib` — Enumerate any PyPI packages or internal services that `import reasons_lib`; this is the sole gate on deletion.
- [general] `deprecation-schedule` — Check whether `reasons_lib/__init__.py` emits a `DeprecationWarning` (it does not, per line 1-3 of the observation) and decide whether one should be added before deletion.
- [file] `CHANGELOG.md:136` — The only historical reference to `reasons_lib.api`; gives the origin context for the shim.

## Beliefs

- `reasons_lib-reexports-reasons` — `reasons_lib/__init__.py:3` and `reasons_lib/api.py:3` are pure `from reasons… import *` shims; no additional logic exists in the legacy package.
- `reasons-requires-no-reasons_lib` — No file in the repository imports `reasons_lib`; the dependency arrow only points from the shim into `reasons/`.
- `reasons_lib-can-be-deleted-without-internal-breakage` — Removing `reasons_lib/` and dropping the entry at `pyproject.toml:31` breaks no code inside this repo.
- `entry-points-use-canonical-module` — Both `project.scripts` entries (`pyproject.toml:27-28`) resolve to `reasons.cli:main` and `reasons.mcp_server:main`, never to `reasons_lib.*`.
- `reasons_lib-shim-has-narrow-surface` — `reasons_lib/` covers only `reasons` top-level and `reasons.api`; any legacy `from reasons_lib.network import …` was already broken before this decision.
