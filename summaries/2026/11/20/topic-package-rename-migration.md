---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Understand why both `reasons/` and `reasons_lib/` exist and which package is intended to be canonical.

# Why both `reasons/` and `reasons_lib/` exist

## Short answer

- **`reasons/` is canonical.** It contains the only real implementation: the `Justification`, `Node`, and `Nogood` dataclasses in `reasons/__init__.py` (59 lines, see observations), plus `network.py`, `api.py`, `storage.py`, `cli.py`, etc. (listed in `CLAUDE.md` lines 22–33).
- **`reasons_lib/` is a legacy alias kept only for backwards compatibility.** Its entire `__init__.py` is 3 lines:

  ```python
  """Backwards-compatibility shim — use ``import reasons`` instead."""
  from reasons import *  # noqa: F401,F403
  ```

## How the two-way state came about

1. **Original name.** `CHANGELOG.md` under `[0.1.0]` records “Python API layer (`reasons_lib.api`) returning dicts…” — this is the pre-rename name of the library package.
2. **Rename in v0.3.0.** The changelog for `[0.3.0]` says: “Renamed from `rms` to `reasons` (CLI, library, package) based on 5pp LLM accuracy improvement in ablation study.”
3. **Compatibility shim left behind.** Rather than break downstream importers, `reasons_lib/__init__.py` was reduced to a one-line re-export from `reasons`.

This history is documented explicitly in `CLAUDE.md`:
- Line 24: `reasons/  # importable as \`import reasons\` (also aliased as reasons_lib for backwards compat)`
- Line 35: `reasons_lib/  # backwards-compat shim — re-exports from reasons/`

## Evidence that `reasons/` is canonical

- **Entry points.** In `pyproject.toml`, both scripts point at the real package:
  - `reasons = "reasons.cli:main"`
  - `reasons-mcp = "reasons.mcp_server:main"`
  Nothing references `reasons_lib.*` for a script entry.
- **Implementation lives in `reasons/`.** The dataclasses (`Justification`, `Node`, `Nogood`) are defined in `reasons/__init__.py`; `reasons_lib/__init__.py` defines nothing of its own.
- **The shim itself says so** — its docstring literally instructs callers to use `import reasons` instead.
- **Developers are told so** — `CLAUDE.md` line 24 labels `reasons_lib` as an alias “for backwards compat.”

## Why `reasons_lib` is still **shipped** at all

`pyproject.toml` line 31:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

Both directories are packaged so that pre-rename importers (`import reasons_lib`) do not break on upgrade. This is a deliberate backwards-compatibility decision, not an architectural one — nothing in the new code depends on `reasons_lib`.

## Caveats / gaps in the observations

- **Submodule re-exports are not faked.** `from reasons import *` only re-exports public names from `reasons/__init__.py`; it does **not** create `reasons_lib.api`, `reasons_lib.network`, etc. Submodule imports like `from reasons_lib.api import …` will only work if `reasons_lib/api.py` still exists on disk. The observations only show `reasons_lib/__init__.py`, but the changelog refers to `reasons_lib.api`, so either a stale file remains or this is a latent break for older callers.
- **Test imports are unaccounted for.** The scan for `(from reasons|import reasons)` returned **0 matches** in the tests. Either tests use relative imports, manipulate `sys.path`, or import via `reasons_lib` — either way, the observations don’t explain how the test suite resolves the canonical package. This is worth verifying directly.

## Guidance for your colleague

- **Write new code against `reasons.*`** (e.g., `from reasons import Node, Justification`, `from reasons.api import ...`).
- **Treat `reasons_lib` as read-only/legacy.** Do not add features to it. It exists only to keep old callers importing successfully.
- **Do not create new `reasons_lib.*` submodules.** Anything new belongs in `reasons/`.
- The day you want to remove `reasons_lib`, you must first confirm no downstream package still imports it and then drop line 31 of `pyproject.toml` along with the directory.

## Topics to Explore

- [file] `reasons_lib/` — Directory listing to confirm whether legacy `api.py`/`network.py` submodules still exist or whether the shim is `__init__.py`-only.
- [file] `tests/` — Where the observed 0 matches for `from reasons`/`import reasons` comes from; the actual import style used by the test suite.
- [file] `reasons/api.py` — The canonical functional API layer that `reasons_lib.api` historically mirrored; useful as the reference for migration.
- [general] `reasons_lib-removal-plan` — Search the whole repo (including downstream projects) for live `reasons_lib` imports to know when the shim can be deleted.
- [function] `reasons/cli.py:main` — The canonical CLI entry point referenced by `pyproject.toml` scripts; shows how the public surface is intended to be consumed.

## Beliefs

- `reasons-is-canonical-package` — `reasons/` contains the actual implementation (dataclasses, network, storage, API, CLI); `reasons_lib/` only re-exports from it.
- `reasons-lib-is-a-backwards-compat-shim` — `reasons_lib/__init__.py` is a 3-line shim docstring plus `from reasons import *`.
- `reasons-lib-shipped-in-distribution` — `pyproject.toml` lists `packages = ["reasons", "reasons_lib"]`, so the shim is wheel-installed on purpose.
- `reasons-rename-created-reasons-lib` — The v0.3.0 `rms → reasons` rename left `reasons_lib` as the legacy import name; it was previously the API layer (`reasons_lib.api` in v0.1.0).
- `reasons-exports-core-dataclasses-from-init` — `reasons/__init__.py` defines the canonical `Justification`, `Node`, and `Nogood` dataclasses.
- `reasons-lib-does-not-reexport-submodules` — `from reasons import *` in the shim does not synthesize submodule aliases like `reasons_lib.api`, so submodule imports only work if separate `reasons_lib/*.py` files still exist on disk.
- `reasons-entry-points-target-reasons-package` — Both `pyproject.toml` scripts (`reasons` and `reasons-mcp`) point at `reasons.cli:main` and `reasons.mcp_server:main`, never at `reasons_lib`.
