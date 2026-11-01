---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/ask.py

# `reasons_lib/ask.py`

## 1. Purpose

This file is a **backwards-compatibility shim**. Its sole job is to let existing code that does `from reasons_lib.ask import ...` continue to work without modification, even though the canonical implementation now lives in `reasons/ask.py`.

The project has two parallel top-level packages — `reasons/` (the real implementation) and `reasons_lib/` (a thin re-export layer). This file is the `ask` module's entry point in that shim layer. The docstring is unambiguous: *"use `from reasons.ask` instead."*

## 2. Key Components

There are none. The file defines no classes, functions, or constants. Its entire body is:

```python
from reasons.ask import *  # noqa: F401,F403
```

- **`from reasons.ask import *`** — Re-exports every public name (anything not starting with `_`, or explicitly listed in `reasons.ask.__all__`) into the `reasons_lib.ask` namespace.
- **`# noqa: F401,F403`** — Suppresses `flake8` warnings:
  - `F401` — "imported but unused" (the names *are* the export; flake8 doesn't know that).
  - `F403` — "star import makes it impossible to tell which names are defined here."

Any name you'd reach for via `reasons_lib.ask` (e.g., `ask`, `AskResult`, or whatever `reasons.ask` exposes) is resolved identically to importing from `reasons.ask` directly.

## 3. Patterns

| Pattern | How it appears here |
|---|---|
| **Backwards-compatibility shim** | A package was renamed/restructured from `reasons_lib` → `reasons`; old import paths are kept alive with star-re-exports so downstream consumers don't break. |
| **Facade re-export** | `reasons_lib` acts as a stable public API surface; the real logic is delegated to `reasons`. Consumers never see a difference at runtime. |
| **Lint suppression for intentional star imports** | The `noqa` directives are the standard idiom for "yes, I know star imports are usually bad; this is a deliberate re-export module." |

Because the shim contains zero logic, there is no branching, no state, and no configuration to manage. Maintenance cost is effectively zero *unless* you delete the `reasons_lib` package, at which point every importer breaks.

## 4. Dependencies

### Imports (outbound)
| Module | What it provides |
|---|---|
| `reasons.ask` | All public symbols of the ask functionality (functions, dataclasses, type aliases, etc.) |

That's the only import. No third-party libraries, no stdlib modules, no relative imports.

### Importers (inbound)
| Importer | Why it matters |
|---|---|
| `reasons/api.py` | The HTTP/API layer likely calls ask-related functions; it imports through this shim (or the test harness resolves names via it). |
| `reasons/cli.py` | CLI entry point wires up the `ask` subcommand. |
| `tests/test_ask.py` | Unit/integration tests for the ask feature. |
| `tests/test_ask_mcp.py` | Tests exercising ask through the MCP (Model Context Protocol) transport. |
| `tests/test_search_sources.py` | Search-sources logic that depends on ask's public API. |

> **Practical note:** Because this is a pure re-export, you can safely replace every `from reasons_lib.ask import X` with `from reasons.ask import X` and the behaviour is identical. The shim exists only so you *don't* have to do that in one big-bang commit.

## 5. Flow

Execution is trivial:

1. Python's import system encounters `reasons_lib.ask`.
2. It executes the single statement `from reasons.ask import *`.
3. Python imports (or reuses the already-loaded) `reasons.ask` module, reads its `__all__` (or all non-underscore names), and binds those names into the `reasons_lib.ask` module's `__dict__`.
4. Any subsequent `from reasons_lib.ask import some_func` resolves `some_func` in that dict and returns the *same* object that lives in `reasons.ask`.

There is no transformation, no wrapping, no lazy loading. The names are identical object references.

## 6. Invariants

- **Identity guarantee:** Every name in `reasons_lib.ask` is the *same* Python object as its counterpart in `reasons.ask`. `isinstance` checks, `id()` comparisons, and `inspect` metadata all agree.
- **No hidden state:** The shim adds no mutable module-level variables, so there's no risk of it diverging from the real module.
- **One-directional dependency:** `reasons_lib.ask` → `reasons.ask`. The reverse is never true, so there is no circular-import risk introduced by this file.

## 7. Error Handling

There is none in this file. Two edge cases to be aware of:

- **If `reasons.ask` doesn't exist or fails to import**, the star import raises the underlying `ImportError` / `ModuleNotFoundError` with the original traceback. The shim adds no wrapping or retry.
- **If a name is removed from `reasons.ask`**, any code that still does `from reasons_lib.ask import that_name` will get an `ImportError` at import time — same as if it imported from `reasons.ask` directly. No silent `None` returns, no deprecation warnings.

## Topics to Explore

- [file] `reasons/ask.py` — The actual implementation this shim delegates to; understand its public API surface, the `ask()` entry point, and any `__all__` it defines.
- [file] `reasons_lib/__init__.py` — Check whether the package-level init re-exports submodules or defines any additional compatibility shims (e.g., module-level aliases).
- [file] `reasons/api.py` — See how the API layer consumes ask functionality and whether it imports through `reasons_lib` or `reasons` directly; this reveals which import path is actually "live" in production.
- [general] `reasons_lib-removal-plan` — Understand the migration strategy: when (if ever) `reasons_lib/` is slated for deletion, and what external consumers (other repos, Hugging Face integrations, MCP clients) might still depend on the old path.
- [file] `tests/test_ask.py` — Read the test suite to understand the expected contract of `ask()`: input shapes, output types, error semantics, and edge cases the implementation must handle.

## Beliefs

- `reasons_lib-ask-is-a-reexport` — Every name accessible via `reasons_lib.ask` is the identical Python object exported by `reasons.ask`; the shim introduces no wrappers, copies, or mutations.
- `reasons-lib-ask-requires-reasons-ask` — Importing `reasons_lib.ask` transitively requires the `reasons` package to be installed and importable; it is not a self-contained module.
- `star-import-produces-identity` — `from reasons_lib.ask import X` and `from reasons.ask import X` yield the same object at runtime, so `isinstance` checks and identity comparisons (`is`) behave identically regardless of import path.
- `reasons-lib-has-zero-logic` — The `reasons_lib` package contains no executable business logic of its own; every module is a pure re-export shim, making the package safe to delete in one atomic commit once all external importers are migrated.
- `noqa-f403-suppresses-star-import-warning` — The `# noqa: F403` directive is load-bearing: without it, any linter configured to enforce `F403` (undefined star imports) would flag this file in CI, blocking merges for a non-issue.
