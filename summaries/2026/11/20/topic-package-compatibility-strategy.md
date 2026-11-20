---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Compare the duplicate `reasons/` and `reasons_lib/` trees and packaging configuration to understand the migration rules.

# Comparing `reasons/` and `reasons_lib/`: Migration Rules

The repository has clearly undergone a rename from `reasons_lib` → `reasons`, and the current state leaves `reasons_lib` behind as a thin backwards-compatibility shim. The observations confirm this in the package layout, `pyproject.toml`, and the git branch names.

## 1. Package layout: skeleton vs. shim

| Aspect | `reasons/` | `reasons_lib/` |
|---|---|---|
| `__init__.py` | 59 lines with dataclasses `Justification`, `Node`, `Nogood` | 3 lines: a docstring + `from reasons import *` |
| `api.py` | Real imports: `copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys` + `collections.abc`, `datetime`, `itertools`, `pathlib`, `.network`, `.storage`, `metadata.build_meta` | `from reasons.api import *` only |
| `merkle.py` | Contains `compute_merkle_hash`, `compute_text_hash`, `compute_content_hash` used by `reasons/network.py` (lines 130, 514, 516) and `reasons/api.py` (lines 2736, 2750) | Not referenced in the observations |

`reasons/` is the **canonical** package. `reasons_lib/` exists only to keep `import reasons_lib` and `import reasons_lib.api` working for old code paths — the docstring literally says `"Backwards-compatibility shim — use import reasons instead."` (`reasons_lib/__init__.py`, line 1). This is corroborated by `CLAUDE.md`:

- Line 24: `reasons/  # importable as import reasons (also aliased as reasons_lib for backwards compat)`
- Line 35: `reasons_lib/  # backwards-compat shim — re-exports from reasons/`

## 2. Packaging configuration (`pyproject.toml`)

Two things matter in `[tool.setuptools]` and `[project.scripts]`:

```toml
[project.scripts]
reasons     = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"

[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

- **Both packages are shipped** as part of the `ftl-reasons` distribution (v0.58.0). `packages = ["reasons", "reasons_lib"]` (line 31) means any user upgrading will still have `reasons_lib` importable, so the shim is a live contract and cannot be deleted in a patch release without a deprecation window.
- **All console entry points target `reasons.*`**, not `reasons_lib.*`. There is no `reasons-lib` (or `reasons_lib-mcp`) script. So invocations of the CLI/MCP server via the installed scripts always exercise the new tree.
- **Dependencies are unchanged** for both packages — `dependencies = []`, plus optional extras (`test`, `pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`). Since both packages ship from one project, there is no version skew between the shim and the implementation.

## 3. Direction of the rename

The rename went `reasons_lib` → `reasons`, not the other way. Evidence:

- Git remote branch: `refs/remotes/origin/rename-reasons-lib-to-reasons` (commit `8a2b1536…`, seen in `.git/packed-refs` line 100 and `.git/FETCH_HEAD` line 99).
- `CLAUDE.md` line 35 calls `reasons_lib` the "backwards-compat shim — re-exports from `reasons/`".
- The old module structure is preserved only in `.forge/proposed-entries.json` entries named `reasons_lib-{__init__,api,cli,network,storage,…}.md` — these are historical doc-generation artifacts for the pre-rename layout.
- `CHANGELOG.md` line 136 still refers to the "Python API layer (`reasons_lib.api`) returning dicts" — historical prose describing the API surface before it was reparented under `reasons.api`.

## 4. Practical migration rules implied by the observations

1. **New code should `import reasons` / `from reasons.api import …`**, matching what the shims forward to and what `[project.scripts]` resolves against.
2. **`reasons_lib.api` and `reasons_lib` are shims only** — they use `import *`, so they only forward names that `reasons/__init__.py` and `reasons/api.py` export. If a name was renamed or made private in `reasons.api`, that name silently disappears from `reasons_lib.api`.
3. **Deep imports are the biggest risk.** The observations only show `reasons_lib/__init__.py` and `reasons_lib/api.py`. Any legacy user doing `from reasons_lib.network import Network`, `from reasons_lib.storage import Storage`, or `from reasons_lib.cli import main` needs a matching shim file present. Given `.forge/proposed-entries.json` shows `reasons_lib-network.md`, `reasons_lib-storage.md`, `reasons_lib-cli.md`, those submodules *existed historically* — but the current filesystem state for them is not in these observations.
4. **Testing must be against the canonical tree.** `tests/test_merkle.py` imports `from reasons.merkle import …` (lines 7, 47, 75, 126). That is the pattern to replicate: tests target `reasons.*`, not `reasons_lib.*`.
5. **Versioning is single-source.** `version = "0.58.0"` governs both installed packages, so the shim and the real package can never drift out of sync.

## 5. Gaps in the observations

- We cannot see whether `reasons_lib/cli.py`, `reasons_lib/network.py`, `reasons_lib/storage.py`, `reasons_lib/merkle.py`, `reasons_lib/compact.py`, `reasons_lib/derive.py`, or `reasons_lib/mcp_server.py` still exist. The `.forge` entries hint they once did. If they are absent, any legacy deep import breaks despite `import reasons_lib` working.
- The scanner reports `from metadata import build_meta` in `reasons/api.py` and `from network import Network`, `from . import Justification`. These are almost certainly relative imports whose leading `.` was stripped — but this is not stated, so the true import paths need confirmation against `reasons/api.py` directly.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — Verify whether it re-exports only `from reasons.api import *` or also explicit names; the shim semantics determine what legacy callers get.
- [file] `reasons_lib/cli.py` — Confirm a CLI shim exists; `.forge/proposed-entries.json` lists `reasons_lib-cli.md`, but `[project.scripts]` only references `reasons.cli:main`, so a missing `reasons_lib.cli` is a real backward-compat break.
- [diff] `refs/remotes/origin/rename-reasons-lib-to-reasons` (commit `8a2b1536…`) — Inspect the actual rename diff to see exactly which submodules moved, which shims were left behind, and which imports were rewritten.
- [general] `reasons_lib-submodule-coverage` — Audit whether `reasons_lib/{network,storage,merkle,compact,derive,mcp_server}.py` still exist; this is the delta between "`import reasons_lib` works" and "all historical `reasons_lib.X` imports work".
- [file] `reasons/api.py` — Confirm the relative-import paths around the `from .network import Network` / `from .storage import Storage` / `from .merkle import …` lines (130 in `network.py`, 2736 & 2750 in `api.py`) that establish `reasons` as the sole canonical implementation.

## Beliefs

- `reasons_lib-is-a-shim` — `reasons_lib/__init__.py` contains only `from reasons import *`, so any name it exposes is sourced from `reasons/__init__.py`.
- `reasons_lib-api-shim-delegates-to-reasons-api` — `reasons_lib/api.py` has no direct imports other than `from reasons.api import *`.
- `entry-points-target-reasons-package-only` — `[project.scripts]` in `pyproject.toml` maps both `reasons` and `reasons-mcp` to `reasons.cli:main` and `reasons.mcp_server:main` respectively; no entry point references `reasons_lib`.
- `pyproject-ships-both-packages` — `[tool.setuptools].packages` in `pyproject.toml` (line 31) lists both `"reasons"` and `"reasons_lib"`, so the shim is installed alongside the canonical package.
- `reasons-replaces-reasons-lib` — The origin branch `rename-reasons-lib-to-reasons` and the `CLAUDE.md` description of `reasons_lib/` as a "backwards-compat shim — re-exports from `reasons/`" establish `reasons` as the replacement package.
- `tests-bind-to-reasons-tree` — `tests/test_merkle.py` imports `from reasons.merkle import …` (lines 7, 47, 75, 126), not `from reasons_lib.merkle import …`, confirming tests validate the canonical tree.
