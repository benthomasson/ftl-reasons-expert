---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Resolve why two near-identical packages exist (and why `merkle.py` is only in `reasons/`) before deep-reading either tree; this determines which files are canonical.

## Short answer

`reasons/` is the canonical implementation. `reasons_lib/` is a backward-compatibility shim kept so old imports such as `import reasons_lib` and `from reasons_lib.api import ...` keep working after the package moved to `reasons`.

Do **not** deep-read both trees as peer implementations. Read `reasons/` first, and inspect `reasons_lib/` only to verify re-export coverage.

### Evidence

- `pyproject.toml` line 31 packages both: `packages = ["reasons", "reasons_lib"]`, but the CLI entry points in lines 27–28 point at the canonical package:
  - `reasons = "reasons.cli:main"`
  - `reasons-mcp = "reasons.mcp_server:main"`
- `CLAUDE.md` lines 24 and 35 explicitly describe the split:
  - `reasons/` is importable as `import reasons` and “also aliased as reasons_lib for backwards compat”
  - `reasons_lib/` is a “backwards-compat shim — re-exports from reasons/”
- `reasons_lib/__init__.py` is only 3 lines:
  - line 2: `from reasons import *`
- `reasons_lib/api.py` has no local imports and only does:
  - `from reasons.api import *`
- `CHANGELOG.md` line 136 references `reasons_lib.api`, which supports the historical-import-path explanation.

So the “two near-identical packages” are not two implementations. They are:

1. `reasons/` — real source of truth.
2. `reasons_lib/` — compatibility wrapper modules that re-export from `reasons/`.

If you see files such as `reasons_lib/network.py` or `reasons_lib/storage.py`, treat them as shims until proven otherwise. The observed pattern is wildcard re-export, not independent logic.

## Why `merkle.py` exists only in `reasons/`

`reasons/merkle.py` is an internal implementation module, not part of the compatibility re-export surface.

Observed Merkle references all target `reasons.merkle`:

- `reasons/api.py` line 2736: `from .merkle import verify_all`
- `reasons/api.py` line 2750: `from .merkle import backfill_hashes as _backfill`
- `reasons/network.py` line 130: `from .merkle import compute_text_hash, compute_content_hash`
- `reasons/network.py` lines 514 and 516 import `compute_content_hash` and `compute_text_hash`
- `tests/test_merkle.py` line 7 imports from `reasons.merkle`

There are no observed references to `reasons_lib.merkle`.

Also, `reasons/__init__.py` defines only the `Justification`, `Node`, and `Nogood` dataclasses. A shim doing `from reasons import *` imports names exposed by `reasons/__init__.py`, not arbitrary submodules like `reasons.merkle`. Therefore `reasons_lib.merkle` would not exist automatically; it would require an explicit shim file such as `reasons_lib/merkle.py` containing `from reasons.merkle import *`. No such file or reference appears in the observations.

The `.forge/proposed-entries.json` entries list `reasons_lib` modules such as `api`, `check_stale`, `cli`, `compact`, `derive`, `import_agent`, `network`, and `storage` (lines 2–14), but no `merkle` entry. This is consistent with `merkle` being internal/canonical-only.

## Practical rule before deep-reading

1. Treat `reasons/` as canonical.
2. Treat `reasons_lib/*` as re-export shims.
3. Follow imports into `reasons.*`; for Merkle hashing, the only canonical file is `reasons/merkle.py`.
4. Use `reasons_lib/` only to check backward-compatible import coverage.
5. Ignore `.forge/proposed-entries.json` as evidence of a second implementation; it appears to be generated/proposed documentation and even it has no `merkle` shim entry.

One caveat: the observations do not show every file under `reasons_lib/`. To confirm every shim, inspect the first lines of any `reasons_lib/*.py` files. They should follow the `from reasons.<module> import *` pattern.

## Topics to Explore

- [file] `reasons/merkle.py` — Canonical implementation of per-justification Merkle hashing; this is the only Merkle module imported by `network.py`, `api.py`, and tests.
- [file] `reasons_lib/api.py` — Confirms the shim pattern for the public API module: no logic, only `from reasons.api import *`.
- [general] `reasons-lib-shim-coverage` — Check which legacy modules have explicit `reasons_lib` shims versus only top-level `from reasons import *`; `merkle` is absent.
- [function] `reasons/merkle.py:compute_merkle_hash` — Entry point for dependency-aware Merkle hash computation; useful for understanding what integrity checking covers.
- [general] `backwards-compatibility-import-paths` — Determine whether downstream callers still import `reasons_lib.*` and whether any missing shim would break them.

## Beliefs

- `reasons-package-is-canonical-implementation` — Implementation and runtime behavior live under `reasons/`; `reasons_lib/` re-exports from it.
- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/__init__.py` uses `from reasons import *`, and `reasons_lib/api.py` uses `from reasons.api import *`.
- `merkle-module-is-canonical-only-in-reasons` — All observed Merkle imports target `reasons.merkle`; there is no observed `reasons_lib.merkle` reference or shim.
- `script-entry-points-use-reasons-not-reasons-lib` — `pyproject.toml` maps `reasons` and `reasons-mcp` to `reasons.cli:main` and `reasons.mcp_server:main`.
- `reasons-init-does-not-export-merkle-submodule` — `reasons/__init__.py` exposes dataclasses only, so `from reasons import *` does not automatically create `reasons_lib.merkle`.
