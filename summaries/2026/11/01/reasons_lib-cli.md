---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/cli.py

# `reasons_lib/cli.py`

## 1. Purpose

This file is a **backwards-compatibility shim** — a one-line re-export that keeps the old `reasons_lib` import path alive after the package was renamed to (or extracted into) `reasons`. Its sole responsibility is to allow existing code written against `reasons_lib.cli` to continue importing without modification. The docstring makes the intent explicit and directs new code to `reasons.cli` instead.

In the broader project, `reasons/` is the canonical package (it's where the actual CLI logic lives), and `reasons_lib/` is a legacy mirror that exists purely so old import statements don't break. Every file in `reasons_lib/` follows this same shim pattern.

## 2. Key Components

There are no classes, functions, or constants defined here. The entire module body is:

```python
from reasons.cli import *  # noqa: F401,F403
```

- **`from reasons.cli import *`** — Pulls every public name (those not starting with `_`, and those listed in `__all__` if defined) from the real implementation into this module's namespace. After this line, `reasons_lib.cli` is effectively a transparent alias for `reasons.cli`.
- **`# noqa: F401`** — Suppresses the "imported but unused" lint warning. The import *is* the module's purpose; it's not "unused."
- **`# noqa: F403`** — Suppresses the "star import" lint warning. Star imports are normally discouraged, but this is the idiomatic way to write a blanket re-export shim.

## 3. Patterns

| Pattern | How it appears here |
|---|---|
| **Compatibility shim / adapter** | The module exposes the same surface as `reasons.cli` without reimplementing anything. |
| **Star re-export** | `import *` guarantees that *all* public symbols are forwarded, so adding a new function to `reasons.cli` automatically makes it available through `reasons_lib.cli` with zero changes here. |
| **Dual-package layout** | The repo ships both `reasons/` and `reasons_lib/` with near-identical file trees. The `reasons_lib` copies are all shims; `reasons` holds the real code. This is a transitional pattern during a package rename. |

## 4. Dependencies

### Outbound (this file imports)
- **`reasons.cli`** — the only dependency. All public names in `reasons.cli` become available through this module.

### Inbound (files that import `reasons_lib.cli`)
The "Imported By" list is extensive and falls into two groups:

- **Internal code still on the old path:** `reasons_lib/mcp_client.py`, and apparently `reasons/cli.py` and `reasons/mcp_client.py` (likely they import a *helper* via the `reasons_lib` path, or the metadata reflects a historical state).
- **Test suite (the bulk):** ~20 test files (`test_cli.py`, `test_ask.py`, `test_derive.py`, `test_repair.py`, `test_roundtrip.py`, `test_verify.py`, etc.) still write `from reasons_lib.cli import …`. These are the primary reason the shim still exists.

> **Key takeaway for maintainers:** You can delete `reasons_lib/` in one shot *only after* every import in the test suite and any remaining `reasons_lib`-prefixed imports in the source tree are rewritten to `reasons.*`.

## 5. Flow

There is no control flow. Execution is:

1. Python imports `reasons_lib.cli`.
2. The single statement `from reasons.cli import *` runs, which:
   a. Imports the `reasons.cli` module (executing its body: argparse setup, command registration, etc.).
   b. Copies every public name into `reasons_lib.cli.__dict__`.
3. The module is cached in `sys.modules['reasons_lib.cli']`.
4. Any subsequent `from reasons_lib.cli import some_func` resolves `some_func` directly from this module's namespace — no further indirection.

There is no runtime dispatch, no lazy loading, no proxy. It's a static namespace copy at import time.

## 6. Invariants

- **Namespace parity:** After import, `set(dir(reasons_lib.cli)) - {'__module__', '__name__', '__doc__', '__package__', ...}` should be a superset of (or equal to) the public names in `reasons.cli`. Any name added to `reasons.cli` is automatically available here.
- **No independent state:** This module defines no variables, classes, or functions of its own. It is a pure view over `reasons.cli`. Modifying this file to add logic would violate the shim contract.
- **Import order safety:** `reasons.cli` must be importable without any `reasons_lib` import (i.e., no circular dependency). The shim goes one direction only: `reasons_lib → reasons`, never the reverse.

## 7. Error Handling

- **If `reasons.cli` fails to import** (missing dependency, syntax error, etc.), the exception propagates upward from this `from … import *` line. There is no `try/except`, no fallback, no logging. The consumer (e.g., a test file) sees the original traceback.
- **If `reasons.cli` removes a name**, any code that did `from reasons_lib.cli import that_name` will get an `ImportError` at *their* import time, not here. This shim doesn't catch or mask missing names.
- **Star-import edge case:** Names starting with `_` are *not* re-exported by `import *` (unless explicitly listed in `__all__`). So private helpers in `reasons.cli` are invisible through this shim, which is the desired behavior.

---

## Topics to Explore

- [file] `reasons/cli.py` — The actual CLI implementation this shim forwards to; understand the argparse commands, subcommand registry, and entry-point structure to know what surface is being re-exported.
- [file] `reasons_lib/__init__.py` — Check whether the package-level shim follows the same pattern and whether `__all__` is defined anywhere, since that governs what `import *` picks up.
- [repo] `reasons/ vs reasons_lib/ diff` — Run a side-by-side comparison to confirm *every* file in `reasons_lib` is a shim and none contain divergent logic; this determines whether deletion is a simple find-and-replace.
- [general] `package-rename-strategy` — The dual-package layout is a transitional state. Understanding the plan (and timeline) for collapsing `reasons_lib` into `reasons` tells you whether to invest in updating the ~20 test imports now or defer.
- [file] `pyproject.toml` — Check the `packages` / `py_modules` configuration to see which package name(s) are actually installed and exposed to `pip`, which determines which import path is "real" for downstream consumers.

---

## Beliefs

- `reasons_lib-cli-is-a-reason-cli-reexport` — `reasons_lib/cli.py` defines zero symbols of its own; its entire namespace is a star-import copy of `reasons.cli`'s public names.
- `shim-deletion-blocked-by-test-imports` — ~20 test files and at least `reasons_lib/mcp_client.py` still import via the `reasons_lib.*` path, so deleting the `reasons_lib/` package will cause `ImportError` in those consumers until they are rewritten.
- `star-import-respects-underscore-convention` — Because the shim uses `import *` (not an explicit name list), any name in `reasons.cli` that starts with `_` is *not* forwarded to `reasons_lib.cli`, preserving the private/public boundary automatically.
- `reasons-lib-has-no-runtime-logic` — The module executes exactly one statement at import time; there is no conditional branching, no lazy evaluation, and no mutable state, so it introduces zero runtime overhead beyond the one-time namespace copy.
