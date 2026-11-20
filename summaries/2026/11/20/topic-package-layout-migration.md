---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Survey `reasons_lib/` vs `reasons/` for other shims like this one; determine the full set of moved modules and the intended end state (which package becomes canonical).

# Survey: `reasons_lib/` vs `reasons/` — Shim Pattern and End State

## What the observations confirm

### The shim pattern

`reasons_lib` is a **backwards-compatibility re-export shim**. Both files observed are 3-line wrappers with an explicit docstring telling callers to migrate:

- `reasons_lib/__init__.py:1` — `"""Backwards-compatibility shim — use ``import reasons`` instead."""`
- `reasons_lib/__init__.py:2` — `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py:1` — `"""Backwards-compatibility shim — use ``from reasons.api`` instead."""`
- `reasons_lib/api.py:2` — `from reasons.api import *  # noqa: F401,F403`

Each shim simply re-exports the entire public surface of the corresponding canonical module via `import *`.

### The canonical implementation lives in `reasons/`

By contrast, `reasons/__init__.py` (59 lines) and `reasons/api.py` (5,886 lines) are the real implementations. `reasons/__init__.py` defines the dataclasses `Justification`, `Node`, and `Nogood` directly, and `reasons/api.py` contains the full functional API (helpers like `_is_visible`, `_get_node_tags`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`) — no re-export shim language.

### The packaging declares both packages

`pyproject.toml` line 32–33:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

So `reasons_lib` is a first-class installed package, not a local compatibility directory — meaning any downstream user who did `import reasons_lib` before the move will still work.

### Nothing in the repo actually imports `reasons_lib`

`grep_reasons_lib_references` returned **zero matches** across the entire repository. That means:

- No source module imports `reasons_lib.*`.
- No test imports `reasons_lib.*`.
- Only `pyproject.toml` (a `packages = [...]` declaration) and the two shim files themselves reference the name.

The only reason `reasons_lib` still exists is **external consumers** who might still be pinning old imports. Internally, the package is dead weight.

### Canonical direction of the move

Every signal points the same way — `reasons` is canonical:

- `pyproject.toml` entry points: `reasons = "reasons.cli:main"` and `reasons-mcp = "reasons.mcp_server:main"` — the CLI and MCP entry points both point into `reasons`, not `reasons_lib`.
- `reasons/merkle.py` is imported by `reasons/api.py:2736` (`from .merkle import verify_all`), `reasons/api.py:2750`, and `reasons/network.py:130`, and by `tests/test_merkle.py:7` — again all under `reasons`.
- The shim docstrings themselves state: "use `import reasons` instead" and "use `from reasons.api` instead".

## The full set of moved modules — what we can and cannot say

### Confirmed moved modules (observed)

- `reasons.api` → shimmed at `reasons_lib/api.py` (3-line wrapper).
- The top-level `reasons` package (`Justification`, `Node`, `Nogood`, etc.) → shimmed at `reasons_lib/__init__.py` (3-line wrapper).

### Not determinable from observations

The observations **only show two `reasons_lib/*.py` files**: `__init__.py` and `api.py`. We do **not** have a directory listing of `reasons_lib/`, so we cannot enumerate the full set of moved modules. Based on the shape of `reasons/`, likely candidates for additional shims include:

- `reasons_lib/network.py` → `reasons.network` (referenced by `reasons/api.py:20` as `from .network import Network`)
- `reasons_lib/storage.py` → `reasons.storage` (referenced by `reasons/api.py:21` as `from .storage import PROPOSALS_SCHEMA, Storage`)
- `reasons_lib/merkle.py` → `reasons.merkle` (referenced in `reasons/api.py:2736` and `reasons/network.py:130`)
- `reasons_lib/metadata.py` → `reasons.metadata` (referenced in `reasons/api.py:19`)
- `reasons_lib/pubsub.py` → `reasons.pubsub` (referenced in `reasons/api.py:18`)
- `reasons_lib/cli.py` → `reasons.cli` (declared in `pyproject.toml` entry point)
- `reasons_lib/mcp_server.py` → `reasons.mcp_server` (declared in `pyproject.toml` entry point)

But these are inferences from the `reasons/` package layout, **not** from observed `reasons_lib/` contents. To determine the actual set, run a directory listing of `reasons_lib/` and diff the file names against `reasons/`.

## The intended end state

**`reasons` is canonical; `reasons_lib` is on a deprecation path toward deletion.**

The signal is unambiguous:

1. The shim docstrings name `reasons` as the intended import path (`reasons_lib/__init__.py:1`, `reasons_lib/api.py:1`).
2. Every entry point and every in-repo import targets `reasons`.
3. `reasons_lib` is unreferenced anywhere except `pyproject.toml` and its own files — it is a pure outbound compatibility surface for external callers.

The likely "done" state: `reasons_lib` is **removed entirely** (both from `pyproject.toml`'s `packages` list and from disk) once enough release cycles have passed that external consumers have migrated. Until then, `reasons_lib` remains a thin, throwaway re-export layer with no logic of its own.

## Topics to Explore

- [file] `reasons_lib/` — Directory listing is required to enumerate every shim module; the observations only surfaced `__init__.py` and `api.py`.
- [file] `pyproject.toml` — The `packages = ["reasons", "reasons_lib"]` line and script entry points are the authoritative source for what's shipped and which package is canonical.
- [general] `alternate-shim-scan` — Run a tree-walk over `reasons_lib/*.py` and grep for `from reasons` to enumerate the actual shim set, since the observation grep for `reasons_lib` only found usage, not shim contents.
- [general] `deprecation-removal-timeline` — Check git history / issue tracker for a planned `reasons_lib` removal (version, date, or deprecation-warning addition) to confirm the end state is removal rather than permanent dual maintenance.
- [general] `downstream-importers` — Search external packaging/dependency metadata off-repo to gauge how many consumers still rely on `reasons_lib`, which determines how much longer the shim must ship.

## Beliefs

- `reasons_lib-is-a-shim-package` — Every observed file under `reasons_lib/` (`__init__.py`, `api.py`) is a 3-line `from reasons... import *` re-export with a "Backwards-compatibility shim" docstring.
- `reasons-is-canonical-package` — Both `pyproject.toml` entry points (`reasons = "reasons.cli:main"`, `reasons-mcp = "reasons.mcp_server:main"`) and every in-repo import target the `reasons` package, not `reasons_lib`.
- `reasons_lib-has-no-internal-dependents` — A repo-wide grep for `reasons_lib` returns zero matches outside `pyproject.toml` and the shim files themselves, so no source or test module imports it.
- `pyproject-packages-includes-reasons_lib` — `pyproject.toml` declares `packages = ["reasons", "reasons_lib"]` at lines 32–33, so `reasons_lib` is installed for external consumers despite having no internal uses.
- `reasons_lib-api-shim-maps-to-reasons-api` — The shim at `reasons_lib/api.py` re-exports `reasons.api` (a 5,886-line module), not the reverse, establishing `reasons.api` as the canonical location for the functional API.
