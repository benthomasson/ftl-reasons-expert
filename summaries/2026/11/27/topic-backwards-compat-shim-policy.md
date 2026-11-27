---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Decide whether `reasons_lib` should be (a) deleted, (b) a full `from reasons.X import *` fanout including `merkle`, `api`, `storage`, `network`, or (c) documented as names-only (equivalent to `from reasons import *` today).

## Decision

Recommend **(c) documented as names-only** — keep `reasons_lib` as a thin backwards-compatibility shim that is equivalent to `from reasons import *`, but explicitly document that it does **not** fan out submodules such as `api`, `storage`, `network`, or `merkle`.

Do **not** turn it into a full wildcard fanout now. If you later confirm there are no external consumers and you are willing to make a breaking release, then **(a) delete** is the clean end state.

### Why (c), not (b)

- `reasons_lib/__init__.py:1-2` is already exactly a names-only shim:
  - line 1: backwards-compatibility docstring
  - line 2: `from reasons import *`
- `reasons/__init__.py:1-59` currently defines only the top-level model types `Justification`, `Node`, and `Nogood`. It does not export `api`, `storage`, `network`, or `merkle`.
- Existing submodule compatibility is handled lazily by separate shim files:
  - `reasons_lib/api.py:1-2` does `from reasons.api import *`
  - `reasons_lib/storage.py:1-2` does `from reasons.storage import *`
- A full fanout in `reasons_lib/__init__.py` would change the compatibility contract from “top-level names only” to “also eagerly import and re-export several submodules.” That can:
  - force import-time loading of optional modules
  - create name collisions between top-level and submodule exports
  - make minimal installs depend on optional extras

That risk is especially relevant because `pyproject.toml:14-24` declares many optional dependencies such as `pg`, `api`, `vertex`, `langfuse`, and `llm`.

### Why (c), not (a) immediately

Deletion is internally tempting because the provided scans report zero `reasons_lib` references in Python and docs. See the `total_count: 0` entries for `reasons_lib_usage_in_python` and `reasons_lib_usage_in_docs`.

However:

- `pyproject.toml:31` explicitly packages `reasons_lib`:  
  `packages = ["reasons", "reasons_lib"]`
- `reasons_lib/__init__.py:1` explicitly says it is a “Backwards-compatibility shim.”
- `reasons_lib/api.py` and `reasons_lib/storage.py` exist as submodule shims. Deleting `reasons_lib` would break imports like `from reasons_lib.api import ...` and `from reasons_lib.storage import ...`.

So the safe move is to keep it documented and deprecate it, then delete in a future breaking release once external usage is confirmed absent.

### Caveat / missing information

The provided import-pattern scan is slightly suspicious: `reasons_lib/__init__.py:2` itself contains `from reasons import *`, yet the `^(from reasons|import reasons)` scan reports zero matches. That suggests the scan may have excluded the shim itself or used a limited scope. Before deleting anything, re-run a proper consumer search and check external package usage. Also verify whether `reasons_lib/merkle.py` and `reasons_lib/network.py` actually exist; the observations only show representative `api.py` and `storage.py` shims.

## Topics to Explore

- [file] `pyproject.toml` — Check the explicit `packages = ["reasons", "reasons_lib"]` entry and optional dependency groups before changing or deleting the shim.
- [file] `reasons/__init__.py` — Confirm the actual top-level public names exported by `from reasons import *`.
- [file] `reasons_lib/api.py` — See the lazy submodule shim pattern that avoids importing optional API dependencies at package import time.
- [general] `external-consumer-usage` — Search package indexes, downstream repos, or release notes to confirm whether any external users import `reasons_lib`.
- [general] `deprecation-policy` — Decide the version boundary and migration messaging for eventually removing the compatibility shim.

## Beliefs

- `reasons-lib-init-reexports-reasons-names` — `reasons_lib/__init__.py:2` imports only the top-level names exposed by `reasons/__init__.py`.
- `reasons-lib-api-is-lazy-submodule-shim` — `reasons_lib/api.py:2` imports `reasons.api` only when the `reasons_lib.api` submodule is imported.
- `reasons-lib-storage-is-lazy-submodule-shim` — `reasons_lib/storage.py:2` imports `reasons.storage` only when the `reasons_lib.storage` submodule is imported.
- `reasons-lib-is-explicitly-packaged` — `pyproject.toml:31` lists `reasons_lib` in the setuptools package list.
- `reasons-lib-has-no-reported-internal-consumers` — The provided scans report zero `reasons_lib` references in Python and docs.
