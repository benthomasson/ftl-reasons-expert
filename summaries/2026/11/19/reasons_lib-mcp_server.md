---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/mcp_server.py

# `reasons_lib/mcp_server.py`

```python
"""Backwards-compatibility shim — use ``from reasons.mcp_server`` instead."""
from reasons.mcp_server import *  # noqa: F401,F403
```

Two lines: a docstring and a wildcard re-export. **This file defines zero functions, classes, or constants of its own.** Everything it appears to contain is borrowed from `reasons.mcp_server` at import time.

---

## 1. Purpose

The repository ships two parallel packages with identical layouts: `reasons/` (canonical) and `reasons_lib/` (legacy name). `reasons_lib/mcp_server.py` is the legacy entry point for the MCP server module. Its only job is to keep `import reasons_lib.mcp_server` (and `from reasons_lib.mcp_server import <thing>`) working for downstream code that predates the rename, while the actual implementation lives once — in `reasons/mcp_server.py`.

The docstring is the migration instruction: *use `from reasons.mcp_server` instead*. This is a "tombstone with a forwarding address" — it owns **namespace continuity**, not behavior.

## 2. Key Components

There are no local definitions. The only "component" is the import binding, and its contents are determined by `reasons/mcp_server.py`:

| Aspect | Behavior |
|---|---|
| Names exposed | Every public (non-underscore) name in `reasons.mcp_server`, or exactly the contents of its `__all__` if that module defines one |
| Object identity | **Preserved.** `reasons_lib.mcp_server.MCPServer is reasons.mcp_server.MCPServer` → `True`. Same for functions, instances, exceptions |
| Module identity | **Not preserved.** `reasons_lib.mcp_server is reasons.mcp_server` → `False`. Two distinct entries in `sys.modules` |
| `__doc__` of this module | The shim's own one-line docstring — the real module's docstring is **not** copied |
| `__file__`, `__name__` | Point at the shim: `.../reasons_lib/mcp_server.py`, `"reasons_lib.mcp_server"` |

Notable consequences of using `import *` rather than an explicit `= ...` assignment list:

- **Underscore names are dropped.** If `reasons.mcp_server` exposes `_register_tool` or `_SERVER`, `reasons_lib.mcp_server._register_tool` raises `AttributeError`. Direct legacy imports of private helpers will break.
- **`__all__` is not re-exported.** `reasons_lib.mcp_server.__all__` does not exist (unless the real module bizarrely listed `"__all__"` in its own `__all__`). A downstream `from reasons_lib.mcp_server import *` therefore falls back to "all non-underscore names in the shim's `__dict__`", which happens to be a near-equivalent set — but the two modules no longer *declare* the same public API.
- **Incidental imports leak.** If `reasons/mcp_server.py` does `import json` or `from mcp.server.fastmcp import FastMCP`, then `reasons_lib.mcp_server.json` and `reasons_lib.mcp_server.FastMCP` are valid attributes. This makes the shim's surface area a superset of the real module's *intended* API.

## 3. Patterns

- **Shim / alias module (Facade with no behavior).** The classic "compatibility package" pattern: ~30 sibling files in `reasons_lib/` presumably follow the same two-line shape, making `reasons_lib` a pure alias tree. Cost: one extra `sys.modules` entry per module; benefit: a rename with no downstream breakage.
- **Wildcard re-export (`from X import *`).** Chosen for zero maintenance — new public symbols in `reasons/mcp_server.py` automatically appear in `reasons_lib/mcp_server.py` with no edit here. The tradeoff is that the API surface is implicit and invisible to static analysis (`F403` is suppressed, not solved).
- **Lint suppression as documentation.** `# noqa: F401,F403` says explicitly: *unused import (F401) and star import (F403) are intentional here*. Without it, any flake8/ruff run would flag this file.
- **Sibling mirroring.** `reasons/` and `reasons_lib/` are directory-for-directory identical in the tree listing, which is the dependency rule the shim implies: `reasons_lib/*` may import `reasons/*`, never the reverse.

## 4. Dependencies

**Depends on (upstream):**
- `reasons.mcp_server` — the sole import. Transitively this pulls in whatever that module needs (presumably the `mcp` SDK / `FastMCP`, plus `reasons.storage`, `reasons.api`, etc.). So **the shim inherits every optional-dependency requirement of the real module**.

**Depended on by (downstream):**
- `tests/test_mcp_server.py` — the only listed importer. This is important: the canonical `reasons/mcp_server.py` has no direct test listed, so the compatibility shim is the *test entry point* for the whole MCP server surface. Deleting this file without editing the test collection breaks `tests/test_mcp_server.py` collection entirely.

**Packaging constraint:** `reasons_lib` must actually be installed/shipped (see `pyproject.toml`) for the shim to do anything; and `reasons` must be shipped *alongside* it, otherwise the shim raises `ModuleNotFoundError: No module named 'reasons'` — arguably worse than the original import error it was meant to prevent.

## 5. Flow

Import resolution for `import reasons_lib.mcp_server`:

1. Python imports the `reasons_lib` package → executes `reasons_lib/__init__.py`.
2. Python locates and executes `reasons_lib/mcp_server.py`.
3. Line 2 triggers `import reasons.mcp_server`:
   - If absent from `sys.modules`, the real module executes top-to-bottom — **including all module-level side effects** (constructing the `FastMCP`/server object, registering tools/resources, reading env vars, etc.).
   - If already imported (e.g., `reasons.api` pulled it in), the cached module is reused and side effects do **not** re-run. This is the standard one-execution-per-process guarantee of `sys.modules`.
4. The `IMPORT_STAR` opcode: if `reasons.mcp_server.__all__` exists, iterate it; otherwise iterate `vars(reasons.mcp_server)` and filter `name.startswith("_")`. Bind each name into this module's `__dict__`.
5. Import completes. There is no further control flow: no `if __name__ == "__main__"` block, no `main()`, no lazy `__getattr__`. **This file cannot be executed as a script meaningfully** — running `python reasons_lib/mcp_server.py` would just bind names and exit.

Data transformation: none. The "transformation" is purely symbolic — `reasons_lib.mcp_server.__dict__` becomes a filtered *copy of the bindings*, not a reference to the other module's dict. This one-way copying is the root of the mutation caveat below.

## 6. Invariants

1. **`reasons.mcp_server` must be importable.** The shim has no fallback, no `try/except ImportError`, no lazy proxy. If the upstream module fails to import (missing `mcp` dependency, config error raised at module scope), this file fails identically — the failure just surfaces with an extra frame pointing at the shim.
2. **No cycles into `reasons_lib`.** Because this file imports `reasons.mcp_server`, any `reasons.*` module that imports `reasons_lib.*` at module scope creates a real cycle risk (partially-initialized module → `ImportError`/`AttributeError` on partially bound names). The direction is strictly `reasons_lib → reasons`.
3. **Object identity is preserved, binding identity is not.** `isinstance(x, reasons_lib.mcp_server.SomeError)` works because it's the same class object. But:
   ```python
   monkeypatch.setattr("reasons_lib.mcp_server.build_server", fake)   # patches the SHIM's binding only
   ```
   The real `reasons.mcp_server.build_server` — and any internal caller inside `reasons/mcp_server.py` that resolves the global in *its own* namespace — is untouched. Tests must patch `reasons.mcp_server.*`.
4. **Private symbols are unreachable through the shim.** `_`-prefixed names are a hard boundary of `import *`.
5. **New public names appear automatically; renames do not.** Renaming a public symbol in `reasons/mcp_server.py` is a silent breaking change for `reasons_lib` consumers — no shim edit, no review signal, no lint warning.
6. **Star-import chains stay roughly closed.** `from reasons_lib.mcp_server import *` re-exports the same public set (since the shim declares no `__all__`), so double-hop wildcard imports do not silently lose symbols — only underscore-prefixed ones.

## 7. Error Handling

There is **none**, deliberately.

- No `try/except`: neither `ImportError` (missing optional dependency of the MCP SDK), nor `ModuleNotFoundError` (`reasons` not installed), nor any exception raised during `reasons.mcp_server` module-level initialization is caught, wrapped, or translated.
- No validation: the shim does not check that specific required symbols exist. If `reasons/mcp_server.py` is refactored to drop the server class, `import reasons_lib.mcp_server` still *succeeds*; the failure is deferred to the consumer's first attribute access → `AttributeError: module 'reasons_lib.mcp_server' has no attribute 'MCPServer'`. That late, opaque failure is the main debugging cost of wildcard shims.
- No warnings: no `DeprecationWarning` is emitted on legacy imports. There is currently **no telemetry telling you who still imports `reasons_lib`**, so you can't safely delete the shim tree based on runtime evidence — only by grepping and by editing `tests/test_mcp_server.py`.

---

## Topics to Explore

- [file] `reasons/mcp_server.py` — The real implementation: which server object, tool/resource registrations, and module-level side effects this shim triggers on import.
- [file] `reasons_lib/__init__.py` — Confirms whether `reasons_lib` is a wholesale alias tree or a partially hand-written package; determines whether `from reasons_lib import mcp_server` resolves to this shim.
- [file] `tests/test_mcp_server.py` — The only known consumer; check whether it imports from `reasons_lib` or `reasons`, and whether `monkeypatch.setattr` targets the shim (a latent bug given binding-copy semantics).
- [file] `pyproject.toml` — Whether both `reasons` and `reasons_lib` are packaged, and which one is the declared top-level import surface for users.
- [general] `star-import-reexport-pitfalls` — How `import *` breaks `monkeypatch`, `__all__` propagation, and private-name access; useful for auditing the other ~30 shims in `reasons_lib/`.

## Beliefs

- `reasons-lib-mcp-server-is-a-pure-shim` — The file defines no functions, classes, or constants; its entire contents are one `from reasons.mcp_server import *` statement.
- `reasons-lib-mcp-server-requires-reasons-mcp-server` — Importing this module fails with the same exception (e.g. `ModuleNotFoundError` for the `mcp` SDK) as importing `reasons.mcp_server` directly, since there is no fallback.
- `wildcard-reexport-omits-underscore-names` — Names beginning with `_` in `reasons/mcp_server.py` are not accessible via `reasons_lib.mcp_server`.
- `wildcard-reexport-drops-all-attribute` — `reasons_lib.mcp_server` has no `__all__` attribute even if `reasons.mcp_server` declares one.
- `shim-preserves-object-identity-but-not-binding-identity` — `reasons_lib.mcp_server.X is reasons.mcp_server.X` holds for public `X`, but `monkeypatch.setattr` on the shim does not affect the real module.
- `importing-shim-triggers-server-registration-side-effects` — Any import of `reasons_lib.mcp_server` executes `reasons/mcp_server.py`'s module scope (server construction, tool registration) exactly once per process.
- `test-mcp-server-depends-on-shim` — `tests/test_mcp_server.py` is the only listed importer of this file, so the canonical `reasons/mcp_server.py` is currently exercised through the alias.
