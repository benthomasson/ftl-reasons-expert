---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/cluster.py

# `reasons_lib/cluster.py`

## 1. Purpose

This file is a **backwards-compatibility shim**. It exists solely so that any code importing from the `reasons_lib.cluster` namespace continues to work unchanged. All real logic lives in `reasons/cluster.py` (the canonical module). The `reasons_lib/` directory mirrors every module in `reasons/` with identical filenames, but each `reasons_lib/*` file is nothing more than a star-import re-export of its `reasons/*` counterpart.

The docstring makes the intent explicit: *"Backwards-compatibility shim — use `from reasons.cluster` instead."* If you are writing new code, you should import from `reasons.cluster`. This shim is there to absorb the break for any downstream consumers (older scripts, external tools, pinned test fixtures) that still reference the old `reasons_lib` package name.

## 2. Key Components

There are **no** locally-defined classes, functions, or constants. The entire surface is one statement:

```python
from reasons.cluster import *  # noqa: F401,F403
```

- **`import *`** — pulls every public name (no leading underscore) defined in `reasons/cluster.py` into the `reasons_lib.cluster` namespace. Whatever functions, classes, and constants `reasons/cluster.py` exports become available here with the same names.
- **`# noqa: F401,F403`** — silences two Flake8/pycodestyle warnings:
  - `F401` – "module imported but unused" (the star import *is* the module's purpose, but a naive linter doesn't know that).
  - `F403` – "star import makes it impossible to tell which names are defined" (acknowledged; the module intentionally hides its surface behind the source module's).

The contract is: **`reasons_lib.cluster.X` is always identical to `reasons.cluster.X` for every public `X`.** There is no transformation, no versioning, no conditional logic.

## 3. Patterns

- **Facade / Delegate pattern.** `reasons_lib.cluster` acts as a thin facade over `reasons.cluster`. It owns no state and performs no logic; it simply forwards the entire public API. This is the minimal form of the delegate pattern.
- **Package rename shim.** The `reasons_lib/ → reasons/` rename was done by keeping the old package alive as a series of one-liners rather than deleting it and breaking consumers. This is a common migration strategy: ship the shim in one release, remove it in a later one.
- **Star-import re-export.** Rather than listing each name (`from reasons.cluster import cluster, merge, …`), the file uses `*`. This means it is **brittle to `__all__` changes** in `reasons/cluster.py` but robust to *additions* (new public names automatically appear in the shim).
- **Mirror directory.** Every file in `reasons/` has a corresponding file in `reasons_lib/` with the same one-line pattern. This is a project-wide convention, not specific to `cluster`.

## 4. Dependencies

| Direction | Module | Relationship |
|-----------|--------|--------------|
| **Imports** | `reasons.cluster` | Sole dependency. All public names are pulled in. |
| **Imported by** | `reasons/api.py`, `reasons/cli.py`, `reasons/cluster.py`, `reasons/contradictions.py`, `reasons/derive.py` | These modules (or their `reasons_lib` counterparts) may `from reasons_lib.cluster import …` for historical reasons. |
| **Imported by (tests)** | `tests/test_api.py`, `tests/test_cli.py`, `tests/test_cluster.py`, `tests/test_contradictions.py`, `tests/test_derive.py` | Test fixtures and test bodies reference the `reasons_lib` path. |

Note the interesting circular-looking entry: `reasons/cluster.py` is listed as both the *imported-by* source and an *importer* of `reasons_lib/cluster.py`. This likely means `reasons/cluster.py` itself does a defensive `import reasons_lib.cluster` (perhaps to verify the shim is wired correctly, or for a lazy-reference / monkeypatch path). Worth verifying in the canonical file.

## 5. Flow

Execution is trivial:

1. Python's import machinery loads `reasons_lib/cluster.py`.
2. It executes `from reasons.cluster import *`, which first triggers the full import (and execution) of `reasons/cluster.py` if it hasn't been imported yet.
3. Every public name in `reasons.cluster.__dict__` is copied into `reasons_lib.cluster.__dict__`.
4. The module is registered in `sys.modules` under both `reasons_lib.cluster` and `reasons.cluster`.
5. Subsequent `from reasons_lib.cluster import some_func` resolves directly from the already-populated dict — no further import work.

There is no runtime logic, no conditionals, no data transformation. The only "flow" is the one-time import-time name copying.

## 6. Invariants

- **Identity guarantee:** For any public name `X`, `reasons_lib.cluster.X is reasons.cluster.X` (same object, not a copy). This is because star-import binds the same object reference.
- **No new public names:** This module must never define its own top-level names. Adding a local function or constant would create a divergence between the two packages and break the "pure delegate" contract.
- **Mirror completeness:** Every module in `reasons/` must have a corresponding shim in `reasons_lib/`. If a new module is added to `reasons/` without a matching `reasons_lib/` file, code that imports the `reasons_lib` variant will get an `ImportError`.
- **`__all__` sensitivity:** If `reasons/cluster.py` defines `__all__`, the star import only re-exports names in that list. If it does not define `__all__`, all names not starting with `_` are re-exported. The shim's visible API is therefore **entirely governed by `reasons/cluster.py`'s own conventions.**

## 7. Error Handling

There is **no explicit error handling** in this file. All errors propagate from the underlying `reasons.cluster` import:

- **`ImportError` / `ModuleNotFoundError`:** If `reasons` is not installed or `reasons/cluster.py` is missing, the star import raises and the `reasons_lib.cluster` module fails to load.
- **Import-time exceptions in `reasons/cluster.py`:** If the canonical module raises during its own top-level execution (e.g., a bad constant computation, a failed `from … import …`), that exception bubbles up through the star import. The shim adds no `try/except`, no logging, no fallback.
- **Silent name loss:** If `reasons/cluster.py` renames a public symbol or moves it behind an underscore, the star import silently drops it from `reasons_lib.cluster`. No warning is emitted. This is the primary maintenance risk of the `import *` approach.

---

## Topics to Explore

- [file] `reasons/cluster.py` — The canonical implementation. All the functions, classes, and logic that this shim re-exports actually live here. You need to read this to understand what the public API surface actually is.
- [file] `reasons_lib/__init__.py` — Check whether the `reasons_lib` package's init does any additional wiring (e.g., registering submodules, setting `__version__`, or doing a bulk star-import) that affects how the shim is consumed.
- [repo] `reasons/` vs `reasons_lib/` mirror convention — Whether *every* module in `reasons_lib/` follows the identical one-line pattern, or if some have diverged. A quick `grep -r "import \*" reasons_lib/` will confirm the pattern's uniformity and surface any exceptions.
- [general] `shim-removal-plan` — The docstring says "use `from reasons.cluster` instead," implying the shim is transitional. Look for a tracked issue, changelog entry, or GitHub workflow that documents *when* and *how* `reasons_lib/` will be removed, so you don't accidentally introduce new `reasons_lib` imports.
- [diff] `pyproject.toml` — Check how the package is declared (single `reasons` package, or both `reasons` and `reasons_lib` as top-level packages). This determines whether both are importable from a downstream consumer's perspective or whether one is an internal alias.

---

## Beliefs

- `reasons_lib-cluster-is-a-reason-cluster-delegate` — `reasons_lib/cluster.py` defines zero local names; its entire public API is a 1:1 re-export of `reasons/cluster.py` via star import, so `reasons_lib.cluster.X is reasons.cluster.X` for every public `X`.
- `reasons-lib-requires-reason-cluster-import` — If `reasons` is uninstalled or `reasons/cluster.py` raises at import time, `import reasons_lib.cluster` fails with the same exception; the shim provides no fallback or graceful degradation.
- `shim-star-import-tracks-underscores` — The shim's visible name set is governed entirely by `reasons/cluster.py`'s `__all__` (or its public-name convention); any rename to a leading-underscore prefix in the canonical module silently removes that name from `reasons_lib.cluster` with no warning.
- `reasons-contains-reasons-lib-mirror` — The repository maintains a full directory-level mirror (`reasons_lib/` for every module in `reasons/`), meaning adding a new module to `reasons/` without a matching `reasons_lib/` shim will break any consumer that imports the `reasons_lib` variant.
