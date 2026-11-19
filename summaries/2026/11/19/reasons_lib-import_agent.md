---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/import_agent.py

# `reasons_lib/import_agent.py`

```python
"""Backwards-compatibility shim — use ``from reasons.import_agent`` instead."""
from reasons.import_agent import *  # noqa: F401,F403
```

Two lines, zero logic. Everything below is about what those two lines imply.

## 1. Purpose

The repo carries **two parallel packages with identical file lists**: `reasons/` and `reasons_lib/`. `reasons/` is the canonical namespace; `reasons_lib/` is the legacy namespace retained so that downstream code (and older imports in the same repo) don't break after the rename.

This file is the legacy path for the agent-import functionality. It owns **no responsibility** — no parsing, no storage, no validation. Its only job is to make `import reasons_lib.import_agent` (or `from reasons_lib.import_agent import X`) resolve to the real implementation in `reasons/import_agent.py`. It is a pure *forwarding alias*.

The docstring at line 1 is the contract: the canonical import is `from reasons.import_agent import ...`.

## 2. Key Components

There are none in the usual sense — no functions, classes, or constants defined here. The components are inherited via the wildcard:

| Name in `reasons_lib.import_agent` | Origin |
|---|---|
| every name in `reasons.import_agent.__all__` (or every non-underscore name if `__all__` is absent) | `reasons.import_agent` |

Contract implied by the wildcard:

- The public surface of `reasons_lib.import_agent` is **exactly** whatever the canonical module exports at the moment this module is first imported.
- Objects are **identical**, not copies: `reasons_lib.import_agent.X is reasons.import_agent.X` for any exported `X`. So `isinstance` checks, exception types, and class identity all agree across the two namespaces.

## 3. Patterns

- **Shim / alias module (star re-export).** The standard Python idiom for a package rename without a breaking release: the old module path stays importable but delegates.
- **Single-hop delegation, no wrapper functions.** No `def ... : return reasons.import_agent.f(...)` indirection, so there is no call overhead and no signature drift — but also no deprecation warning hook.
- **Lint suppression as policy.** `# noqa: F401,F403` declares "unused import" (F401) and "star import" (F403) as intentional, so a repo-wide flake8/ruff pass doesn't fight the shim.
- **Wildcard export as an implicit API contract.** The shim silently assumes `reasons/import_agent.py` defines `__all__`; if it doesn't, `dir()` of this module will also pick up whatever non-underscore names the canonical module literally imported (e.g. `os`, `json`, `dataclass`) — namespace pollution that then leaks into `from reasons_lib.import_agent import *` in the next hop.

## 4. Dependencies

**Depends on (line 2):**
- `reasons.import_agent` — the canonical implementation. Importing the shim therefore *executes the entire canonical module*, including its transitive imports (`reasons.storage`, `reasons.api`, etc., depending on what it pulls in).

**Imported by (per repo graph):**
- `reasons/api.py` — the *canonical* API module imports the *legacy* shim.
- `reasons/pg.py` — same.

That direction is worth pausing on: the new namespace importing the old one means there is a real import chain

```
reasons.api  →  reasons_lib.import_agent  →  reasons.import_agent  →  (…possibly reasons.api again)
```

Because `reasons_lib/__init__.py` is a package in its own right, this also drags the legacy package into `sys.modules` for every process that touches `reasons.api` or `reasons.pg`. It is the single biggest maintenance risk in this file.

**Depended on by tests:** `tests/test_import_agent.py` exercises the shim path; `tests/test_roundtrip.py` and others may exercise both namespaces.

## 5. Flow

1. Interpreter imports `reasons_lib` → `reasons_lib/__init__.py` runs.
2. `reasons_lib.import_agent` begins execution: docstring assigned to `__doc__`.
3. Line 2 triggers a full import of `reasons` → `reasons/import_agent.py` (running that module top-to-bottom).
4. On success, every name in `reasons.import_agent.__all__` (or every non-underscore key of its `__dict__`) is bound into this module's globals.
5. Module object is cached in `sys.modules['reasons_lib.import_agent']`; subsequent imports are free.

Note that `__doc__` is **not** inherited — the shim keeps its own "Backwards-compatibility shim" docstring, which is correct and useful for `help()`.

## 6. Invariants

- **No state, no side effects** beyond the transitive import of the canonical module. Nothing is cached, mutated, or registered here.
- **Public-only visibility.** Names starting with `_` in `reasons/import_agent.py` are *not* reachable through this path (line 2's wildcard skips them when no `__all__`, and skips them again at the next wildcard hop even when `__all__` lists them). Any internal caller doing `from reasons_lib.import_agent import _helper` is a bug.
- **`__all__` does not propagate.** `from M import *` binds names, not `M.__all__` itself. Unless the canonical `__all__` literally contains `"__all__"`, `reasons_lib.import_agent.__all__` will be absent, so a downstream `from reasons_lib.import_agent import *` falls back to the "all non-underscore names" rule — a subtly different export set from the canonical module.
- **Snapshot, not live view.** The binding happens once, at first import. Names added to `reasons.import_agent` at runtime (monkeypatching in tests, plugin registration) are **not** mirrored; conversely, `reasons_lib.import_agent.X = stub` rebinds only the shim's global and does not affect `reasons.import_agent.X`. Attribute access via `import reasons_lib.import_agent as m; m.X` is a live lookup on the *shim's* dict — so patching the canonical module is invisible through the shim.
- **Ordering constraint:** `reasons.import_agent` must be importable without requiring `reasons_lib` — otherwise step 3 can loop. If `reasons/import_agent.py` imports `reasons.api` (which imports this shim), and `reasons.api` is the entry point, step 3 will observe a half-initialised `reasons.api`.

## 7. Error Handling

None — the module contains no `try`, no validation, no fallback.

- **Failure mode 1: `ModuleNotFoundError: No module named 'reasons'`** — raised at line 2 if the package layout is broken or the distribution is installed without `reasons/`. Since there is no `try/except ImportError` fallback, this error propagates loudly at import time and takes down the whole importing chain (`reasons.api`, `reasons.pg`).
- **Failure mode 2: `ImportError` / `AttributeError: partially initialized module`** — the circular-import hazard described in §4. Symptom is typically `AttributeError: partially initialized module 'reasons.api' has no attribute 'X'` raised from inside `reasons/import_agent.py`, which is confusing because the traceback root is this two-line file.
- **Failure mode 3: silent nothing.** No `DeprecationWarning` is emitted, so there is no telemetry telling you whether the legacy path is still in use. If the goal is to eventually delete `reasons_lib/`, this shim provides no signal.

## Topics to Explore

- [file] `reasons/import_agent.py` — The real implementation behind this shim; check whether it defines `__all__` and whether it imports `reasons.api` at module level (the cycle risk).
- [file] `reasons/api.py` — Uses the legacy shim rather than the canonical module; confirming why reveals whether the `reasons → reasons_lib → reasons.import_agent` chain is intentional or leftover.
- [file] `tests/test_import_agent.py` — Shows which of the two namespaces the tests assert against, and whether object identity across namespaces is verified.
- [general] `package-rename-compat-strategy` — Whether every `reasons_lib/*.py` follows the same shim shape, or whether some still hold the implementation (which would make `reasons/*.py` the shim and invert the dependency direction).
- [general] `deprecation-signalling` — How (or whether) the project warns callers off `reasons_lib` before removal, and what the removal criterion is.

## Beliefs

- `reasons-lib-import-agent-is-a-compat-shim` — `reasons_lib/import_agent.py` defines no functions or classes; its entire body is a docstring plus one wildcard re-export from `reasons.import_agent`.
- `reasons-lib-import-agent-requires-reasons-import-agent` — Importing `reasons_lib.import_agent` necessarily executes `reasons/import_agent.py` in full, including its transitive imports.
- `reasons-api-requires-reasons-lib-import-agent` — `reasons/api.py` and `reasons/pg.py` import the legacy shim, creating a `reasons → reasons_lib → reasons.import_agent` import chain.
- `star-import-drops-private-names` — Any `_`-prefixed symbol in `reasons/import_agent.py` is unreachable through `reasons_lib.import_agent`, because the wildcard import filters underscore names.
- `shim-emits-no-deprecation-warning` — The module contains no `warnings.warn`, so use of the legacy path is invisible at runtime.
- `shim-binds-symbols-by-reference` — Exported objects are the same objects as in the canonical module (`is` identity holds), but rebinding an attribute on either module does not affect the other.

