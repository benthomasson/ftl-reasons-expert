---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How `reasons_lib` → `reasons` was staged, whether any external import paths are still supported, and when/whether the shim can be deleted.

## How `reasons_lib` → `reasons` was staged

The migration used a classic **dual-package + wildcard re-export shim** pattern rather than an in-place rename.

**1. Both packages are still shipped together.** In `pyproject.toml`:

```
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So installing `ftl-reasons` (version `0.58.0`) places both `reasons/` and `reasons_lib/` on `sys.path`. That is what keeps the old import path working for downstream code.

**2. The old package is a pure shim, not an implementation.**

- `reasons_lib/__init__.py` (3 lines total) — docstring plus `from reasons import *  # noqa: F401,F403`.
- `reasons_lib/api.py` (3 lines total) — docstring plus `from reasons.api import *  # noqa: F401,F403`.

The only real implementation lives in `reasons/__init__.py` (59 lines: `Justification`, `Node`, `Nogood` dataclasses) and in `reasons/api.py` (not shown, but referenced by the shim). Nothing in `reasons_lib/` defines any behaviour of its own.

**3. Entry points already target the canonical package.** `[project.scripts]` in `pyproject.toml` points to `reasons.cli:main` and `reasons.mcp_server:main`, not `reasons_lib.*`. So the CLIs were moved to `reasons` and no longer need the shim.

### Are external import paths still supported?

Yes — two are effectively supported today:

| Import path | Backed by | Status |
|---|---|---|
| `import reasons` / `from reasons import Justification, Node, Nogood` | `reasons/__init__.py` | Canonical |
| `from reasons.api import ...` | `reasons/api.py` | Canonical |
| `import reasons_lib` / `from reasons_lib import ...` | `reasons_lib/__init__.py` → `from reasons import *` | Backwards-compat shim |
| `from reasons_lib.api import ...` | `reasons_lib/api.py` → `from reasons.api import *` | Backwards-compat shim |

`CLAUDE.md` line 24 documents this explicitly: *"importable as `import reasons` (also aliased as reasons_lib for backwards compat)"*, and line 35 annotates `reasons_lib/` as *"backwards-compat shim — re-exports from reasons/"*.

Note: wildcard re-exports only carry names that are not underscore-prefixed, so anything private in `reasons/__init__.py` or `reasons/api.py` is **not** visible through the shim. That is acceptable for the public classes (`Justification`, `Node`, `Nogood`), but any internal helper that upstream code briefly imported from `reasons_lib.*` would already be broken — which is a useful signal that only public API was ever intended.

### When/whether the shim can be deleted

The shim can be deleted as soon as you are willing to break third-party code that still imports `reasons_lib`. Evidence in the repo:

- **No internal dependencies.** A search for `reasons_lib` across the whole tree returned `total_count: 0` — nothing inside this repository imports it anymore. Removing it self-hosting-wise is a no-op.
- **Nothing in the CLI or entry points references it.** They all use `reasons.*`.
- **The changelog never documents the rename release explicitly.** `CHANGELOG.md` mentions `reasons_lib.api` only once, at v0.1.0 (line 136), and the only rename it records is `rms` → `reasons` at v0.3.0. There is no line that says *"renamed `reasons_lib` to `reasons`, kept a shim for backwards compat"*. Meanwhile `pyproject.toml` reports `0.58.0` while `CHANGELOG.md` tops out at `0.18.0` — so the rename (and possibly several other changes) is not reflected in the changelog at all.

Because of that, "when" is really a **policy** decision, not a code constraint:

- You are pre-1.0 (`0.58.0`), so semver permits a break, but only if you document it.
- The shim is *silent* — no `DeprecationWarning` is emitted today. A safe removal path is: (a) add a `DeprecationWarning` in each shim, (b) document the removal in the next minor/major changelog, (c) delete `reasons_lib/` **and** drop `"reasons_lib"` from `[tool.setuptools] packages` in `pyproject.toml` in the next breaking release.
- The moment `reasons_lib` is removed from `pyproject.toml`, downstream `pip install ftl-reasons` will shrink and `import reasons_lib` will start raising `ModuleNotFoundError`. Those two edits must land together.

### Gaps in the observations

- The contents of `reasons/api.py` were not provided, so we can't verify what `reasons_lib/api.py`'s wildcard actually re-exports.
- The exact release at which `reasons_lib` → `reasons` happened is not recorded in `CHANGELOG.md` — the top-of-file is v0.18.0 while `pyproject.toml` says 0.58.0. A missing-version gap.
- No tests or warnings exercising `reasons_lib.*` are shown, so we can't confirm whether removal is CI-guarded.

## Topics to Explore

- [file] `reasons/api.py` — Confirm the actual names being wildcard-exported through `reasons_lib/api.py` and whether an `__all__` is defined there.
- [file] `reasons_lib/__init__.py` — Current shim is silent; explore adding a `DeprecationWarning` before deletion.
- [file] `pyproject.toml` — `[tool.setuptools] packages` must drop `"reasons_lib"` in the same change that deletes the directory.
- [general] `downstream-consumer-audit` — Search PyPI dependents / GitHub for `import reasons_lib` before deleting, since the shim is silent.
- [general] `changelog-version-sync` — Reconcile `pyproject.toml` version `0.58.0` with `CHANGELOG.md` topping out at `0.18.0`, and document the `reasons_lib` rename retroactively.

## Beliefs

- `reasons-lib-implements-a-shim` — `reasons_lib/__init__.py` and `reasons_lib/api.py` contain only a docstring plus a `from reasons[.api] import *` line; no implementation lives in `reasons_lib`.
- `both-packages-are-packaged-via-setuptools` — `pyproject.toml` `[tool.setuptools] packages = ["reasons", "reasons_lib"]` means `pip install ftl-reasons` currently installs both.
- `no-internal-code-imports-reasons-lib` — A repo-wide search for `reasons_lib` returned zero matches, so all first-party imports already target `reasons`.
- `cli-entry-points-use-canonical-reasons` — `[project.scripts]` maps `reasons` and `reasons-mcp` to `reasons.cli:main` and `reasons.mcp_server:main`, avoiding `reasons_lib`.
- `reasons-lib-removal-requires-editing-pyproject-toml` — Deleting `reasons_lib/` without also dropping `"reasons_lib"` from `[tool.setuptools] packages` would leave a dangling package reference.
- `changelog-omits-reasons-lib-rename` — `CHANGELOG.md` references `reasons_lib.api` only at v0.1.0 and records `rms`→`reasons` at v0.3.0, but never documents the `reasons_lib` → `reasons` module rename.
