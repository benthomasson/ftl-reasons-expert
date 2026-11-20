---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Explains how the project migrates users from `reasons_lib` to `reasons` without breaking imports.

## How `reasons_lib` users are migrated to `reasons`

The project uses a **backwards-compatible shim package** rather than a codemod or runtime module aliasing. New code should import from `reasons`; old code can keep importing from `reasons_lib` for the paths that have shims.

### What happens on import

- The canonical implementation lives in `reasons/__init__.py`. It defines the public model classes:
  - `Justification` at `reasons/__init__.py:7`
  - `Node` at `reasons/__init__.py:27`
  - `Nogood` at `reasons/__init__.py:53`

- The old top-level package is kept as a real package:
  - `reasons_lib/__init__.py:2` contains `from reasons import *`
  - So `import reasons_lib` and `from reasons_lib import Justification` continue to resolve to the canonical objects from `reasons`.

- Old submodule imports are preserved with per-module shims:
  - `reasons_lib/api.py:2` contains `from reasons.api import *`
  - `reasons_lib/cli.py:2` contains `from reasons.cli import *`
  - This keeps paths like `from reasons_lib.api import ...` and `from reasons_lib.cli import ...` working for the observed shim modules.

### Packaging keeps both names installable

`pyproject.toml:31` explicitly includes both packages:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

That means installing the project installs both the new `reasons` package and the old `reasons_lib` compatibility package.

The console entry points at `pyproject.toml:27-28` only define the new commands:

```toml
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```

So import compatibility is preserved, but old console-script names are not shown as preserved.

### There is no `sys.modules` aliasing

No `sys.modules` matches were found. That means the migration is not implemented by doing something like:

```python
sys.modules["reasons_lib"] = reasons
```

Instead, `reasons_lib` is a distinct package object that re-exports names from `reasons`. In practice:

- `reasons_lib.Node is reasons.Node` should be true because the same class object is re-exported.
- `reasons_lib is reasons` is not true.
- `reasons_lib.api` is a shim module, not literally the same module object as `reasons.api`.

That distinction matters only if user code checks module identity or `__name__`.

### Internal code has already moved to `reasons`

The search for `reasons_lib` in Python files returned **no matches**. So the project itself no longer imports from `reasons_lib`; the shim exists for external consumers.

### Documentation acknowledges the shim

- `CLAUDE.md:24` says `reasons/` is importable as `import reasons`, and is “also aliased as reasons_lib for backwards compat.”
- `CLAUDE.md:35` describes `reasons_lib/` as a “backwards-compat shim — re-exports from reasons/.”
- `CHANGELOG.md:136` mentions the older `reasons_lib.api` API layer.

### Important limitations

- Only `reasons_lib/__init__.py`, `reasons_lib/api.py`, and `reasons_lib/cli.py` are observed as shims. If old code imports another submodule such as `reasons_lib.some_module`, that path would need its own shim and is not shown here.
- No deprecation warning is observed. The old imports keep working silently.
- No `__all__` is shown in the shims, so they rely on wildcard re-export behavior.
- No tests are shown verifying every historical `reasons_lib.*` import path.

In short: the project migrates users by making `reasons` canonical, keeping `reasons_lib` installed, and re-exporting the public API from shim modules so old imports keep working for the observed paths.

## Topics to Explore

- [file] `reasons/__init__.py` — Canonical top-level public API and data model classes that the shims re-export.
- [file] `reasons_lib/api.py` — Example of the submodule shim pattern used to preserve old API imports.
- [file] `pyproject.toml` — Shows both packages are installed and which console entry points are exposed.
- [general] `shim-test-coverage` — Whether tests verify `import reasons_lib`, `from reasons_lib.api import ...`, and other historical import paths.
- [general] `deprecation-policy` — Whether `reasons_lib` imports are expected to emit warnings or have a removal timeline.

## Beliefs

- `reasons-is-canonical-package` — The implementation and public model classes live in `reasons/__init__.py`; `reasons_lib` re-exports them.
- `reasons-lib-requires-reasons` — `reasons_lib/__init__.py:2` depends on `from reasons import *` for top-level compatibility.
- `reasons-lib-api-shim-requires-reasons-api` — `reasons_lib/api.py:2` re-exports from `reasons.api` to preserve old submodule imports.
- `reasons-lib-cli-shim-requires-reasons-cli` — `reasons_lib/cli.py:2` re-exports from `reasons.cli` to preserve old CLI-module imports.
- `both-packages-are-installed` — `pyproject.toml:31` lists both `reasons` and `reasons_lib` in setuptools packages.
- `no-sys-modules-aliasing` — No `sys.modules` aliasing was observed, so `reasons_lib` is a real shim package rather than an alias entry.
