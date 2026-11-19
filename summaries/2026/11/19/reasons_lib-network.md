---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/network.py

# `reasons_lib/network.py` — Backwards-Compatibility Shim

## Purpose

This file is a **one-line compatibility shim**. It exists solely to keep old import paths working after the codebase was reorganized: code that previously did `from reasons_lib.network import X` (or `import reasons_lib.network`) still resolves, but the real implementation lives in `reasons/network.py`.

The file contains no logic of its own — its entire contract is:

> "Re-export every public symbol from `reasons.network` under the same names in `reasons_lib.network`."

It sits in the pattern seen throughout this repo: `reasons_lib/` is a fully-mirrored directory of `reasons/`, and the tests / older consumers still reference `reasons_lib.*`. Compare the pairings in the repository tree (`reasons_lib/api.py` ↔ `reasons/api.py`, `reasons_lib/storage.py` ↔ `reasons/storage.py`, etc.) — this file is one instance of that mass-migration shim.

## Key Components

There are no classes, functions, or constants defined **in this file**. The only element is:

```python
from reasons.network import *  # noqa: F401,F403
```

- `# noqa: F401` — suppresses flake8's "imported but unused" warning. Without it, the linter would flag the star import because no name is referenced locally.
- `# noqa: F403` — suppresses "star import used" warning, since `import *` is generally discouraged. This project deliberately accepts it for shims.

Everything "exported" from `reasons_lib.network` is whatever `reasons/network.py`'s `__all__` (or, absent `__all__`, its non-underscore module-level names) contains. The shim's public surface is therefore **entirely derived** from `reasons/network.py`.

## Patterns

- **Shim / forwarding module**: The canonical migration path in this repo — move the real code to the `reasons` package, leave a star-import stub in `reasons_lib` so downstream imports keep working while the package name is transitioned. It is the same shape as `reasons_lib/__init__.py` likely is.
- **Wildcard re-export as public API mirroring**: Using `from X import *` rather than explicit `from X import (a, b, c)` means the shim automatically inherits new symbols added to `reasons.network`. The trade-off is no explicit contract; `reasons.network.__all__` (if present) is the sole thing controlling what leaks through.
- **Lint suppression via `noqa`**: Instead of restructuring the import, the file locally silences flake8 for exactly the two rules (`F401`, `F403`) that star-import shims trigger. This is the idiomatic fix for this stub and keeps CI green without a global per-file exception.

## Dependencies

**Depends on**
- `reasons.network` — the real module. Every symbol exposed here comes from there. If `reasons.network` is renamed or its `__all__` shrinks, this shim's surface silently shrinks too (no import error, just missing names — this is the risk of star imports).

**Depended on by** (from `Imported By`):
- **Library code inside `reasons/`** — `api.py`, `check_stale.py`, `compact.py`, `export_card.py`, `export_markdown.py`, `import_agent.py`, `import_beliefs.py`, `storage.py`. Note that these `reasons/*` files import from `reasons_lib.network` (the shim) rather than directly from `reasons.network`. That is unusual — it means there is still an internal dependency on the *old* name, likely because those modules were the last not yet updated, or because the intent is to flip the shim direction later.
- **Tests** — the bulk of consumers (`test_network.py`, `test_network_cache.py`, `test_storage.py`, `test_merkle.py`, `test_import_*`, etc.). Tests were written against the historical import path and were intentionally kept on `reasons_lib.network` so that they exercise the shim and catch breakage in the re-export.

## Flow

There is no runtime control flow beyond Python's import machinery:

1. Consumer does `import reasons_lib.network` (or `from reasons_lib.network import X`).
2. Python executes this module top-to-bottom:
   - Triggers `import reasons.network` (the real module), which executes fully — its own imports, class definitions, etc.
   - Copies every public name from `reasons.network`'s namespace into this module's namespace.
3. The consumer resolves `X` from `reasons_lib.network.__dict__`, which is actually the object defined in `reasons.network`.

**Identity consequence:** objects imported through this shim are *the same objects* as through `reasons.network` (same `.py` file, same class identity). This matters for `isinstance`, `pickle`, `is` comparisons, and module-level singletons: `reasons_lib.network.Foo is reasons.network.Foo` is `True`. It also means there is **no duplicate state**; both import paths observe the same globals, so mutations via one path are visible via the other.

## Invariants

This file is small but enforces a few implicitly:

- **Single source of truth**: `reasons/network.py` owns all definitions. This file must never define module-level state or shadow a name — doing so would break the equivalence assumption held by consumers (any `reasons_lib.network.Foo is reasons.network.Foo` assertion in tests would fail).
- **No `__all__` here**: The shim intentionally does not override `__all__`, so it re-exports whatever `reasons.network` chooses to expose. Adding `__all__` here would be a divergence and should be avoided.
- **Directional import**: it must import from `reasons.network`, never the reverse, or a circular import would result (since `reasons.network` is the canonical module and `reasons/*` consumers import through the shim).
- **Runtime equivalence**: any name available as `reasons.network.<name>` should also be retrievable as `reasons_lib.network.<name>` — but only names included in `reasons.network.__all__` / non-underscore names. Names starting with `_` are *not* re-exported by `import *`.

## Error Handling

There is no explicit error handling and no validation here. Error semantics are inherited entirely from the import mechanism:

- If `reasons.network` fails to import (e.g., an `ImportError`, `SyntaxError`, or a dependency like `psycopg`/`httpx` is missing), the failure propagates unchanged from `from reasons.network import *`. The shim does not catch or wrap it.
- If a consumer imports a name that is *not* in `reasons.network.__all__` / public surface, they get a standard `ImportError: cannot import name ...` (or `AttributeError` on attribute access). The shim provides no custom message hinting at the migration; the fix is to look at `reasons/network.py`.
- If `reasons.network` is deleted or renamed, this file raises `ModuleNotFoundError` at import time — the intended, loud signal that the shim is stale.

## Topics to Explore

- [file] `reasons/network.py` — The actual implementation this shim forwards to; check its `__all__` to know the real public surface of `reasons_lib.network`.
- [file] `reasons_lib/__init__.py` — Likely follows the identical shim pattern; useful to confirm the `reasons_lib` → `reasons` migration convention.
- [general] `import-direction-audit` — Investigate why `reasons/api.py`, `reasons/storage.py`, etc. still import from `reasons_lib.network` rather than `reasons.network` directly; that may be a leftover to clean up.
- [file] `tests/test_network_cache.py` — Tests targeting `reasons_lib.network` that likely validate the cache/behavior of the underlying module through the shim; a good place to confirm the `is`-identity assumption holds.
- [general] `package-rename-migration` — Understand the overall migration from `reasons_lib` to `reasons` and the completion criteria for retiring these shims.

## Beliefs

- `reasons-lib-network-is-shim` — `reasons_lib/network.py` defines no code of its own; its only statement is `from reasons.network import *`, forwarding all public names from the canonical module.
- `shim-preserves-object-identity` — Objects imported via `reasons_lib.network` are the same objects as via `reasons.network`, since both paths resolve to the module file `reasons/network.py`.
- `shim-requires-noqa-F401-F403` — The star import is deliberately wrapped in `# noqa: F401,F403`, indicating flake8 is enabled in CI and would otherwise flag the unused/star import.
- `reasons-internal-requires-reasons-lib-shim` — Several `reasons/*` modules (`api.py`, `storage.py`, `compact.py`, `check_stale.py`, `export_card.py`, `export_markdown.py`, `import_agent.py`, `import_beliefs.py`) still import from `reasons_lib.network`, so removing this file would break the canonical package itself, not just tests.
- `shim-exports-only-public-names` — Because `import *` skips underscore-prefixed names and honors `reasons.network.__all__` if defined, private names in `reasons/network.py` are not accessible through the shim.
