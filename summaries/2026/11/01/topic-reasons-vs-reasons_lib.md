---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Topic: The distribution split: comparing `reasons/merkle.py` and `reasons/metadata.py` (present) vs. `reasons_lib/` (absent or different) reveals what's "full" vs. "library"

# The Distribution Split: `reasons/` (Full) vs. `reasons_lib/` (Shim)

## The Short Version

`reasons_lib/` is **not** a separate library with its own logic. It's a two-file backwards-compatibility shim that exists solely so that `import reasons_lib` and `from reasons_lib.metadata import build_meta` don't blow up for people who were using the old import path. The entire implementation lives in `reasons/`.

## What Each Side Actually Contains

### `reasons/` — The "Full" Package

This is where every byte of real code lives. From the observations:

| File | Lines | Role |
|------|-------|------|
| `reasons/__init__.py` | 59 | Data model: `Justification`, `Node`, `Nogood` dataclasses (the core belief-network types) |
| `reasons/merkle.py` | 147 | Merkle-tree hashing for chain integrity — `compute_text_hash`, `_fresh_merkle_hash`, `_fresh_content_hash`, `compute_merkle_hash`, `compute_content_hash`, `verify_node` |
| `reasons/metadata.py` | 56 | Export/storage metadata: `SCHEMA_VERSION`, `_get_generator()`, `build_meta()` |
| `reasons/network.py` | (≥516) | Network operations; imports from `.merkle` at lines 130, 514, 516 |
| `reasons/api.py` | (≥2750) | High-level API; imports `verify_all` and `backfill_hashes` from `.merkle` at lines 2736, 2750 |
| `reasons/cli.py` | (not shown) | CLI entry point (`reasons` script in `pyproject.toml`) |
| `reasons/mcp_server.py` | (not shown) | MCP server entry point (`reasons-mcp` script) |

`pyproject.toml` confirms the full package ships with **zero hard dependencies** (`dependencies = []`) and a rich set of optional extras (`pg`, `cluster`, `mcp`, `api`, `vertex`, `llm`, etc.). The `[tool.setuptools]` section explicitly lists **both** packages:

```toml
[tool.setuptools]
packages = ["reasons", "reasons_lib"]
```

### `reasons_lib/` — The "Library" (i.e., the Shim)

Two files, six lines of actual code total:

**`reasons_lib/__init__.py`** (3 lines):
```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

**`reasons_lib/metadata.py`** (3 lines):
```python
"""Backwards-compatibility shim — use ``from reasons.metadata`` instead."""
from reasons.metadata import *  # noqa: F401,F403
```

That's it. There is **no** `reasons_lib/merkle.py`. There is **no** `reasons_lib/network.py`. There is **no** `reasons_lib/api.py`. The grep for "merkle" across the entire codebase returns 17 matches — every single one is in `reasons/`, `tests/`, or referencing `reasons.merkle`. Zero hits in `reasons_lib/`.

## What This Split Tells You

1. **`reasons_lib` is a migration artifact, not a code layer.** The package was almost certainly renamed from `reasons_lib` → `reasons` at some point (the `pyproject.toml` version is `0.58.0`, suggesting many iterations). Rather than breaking every downstream `import reasons_lib`, the team dropped in star-import shims. The docstrings are explicit: *"use `import reasons` instead."*

2. **The shim is intentionally incomplete.** Because it only re-exports the top-level `__init__.py` (the three dataclasses) and `metadata.py`, any sub-module like `merkle` is **not** reachable via `reasons_lib.merkle`. If you write:
   ```python
   from reasons_lib.merkle import compute_merkle_hash  # ❌ ModuleNotFoundError
   ```
   You get an import error. The "library" side simply doesn't contain that module. The full `reasons/` package is the only path to Merkle hashing, network operations, the API, or the CLI.

3. **The `noqa: F401, F403` comments are load-bearing.** They suppress flake8's "imported but unused" and "wildcard import" warnings. Without them, the shim files would fail lint. This is a signal that these files were written to satisfy a linter, not to define an API surface.

4. **The merkle module is internal to `reasons/`.** Its public consumers are `reasons/network.py` (lines 130, 514–516) and `reasons/api.py` (lines 2736, 2750), plus the test suite (`tests/test_merkle.py`). It's never exposed through the shim. This is the "full vs. library" boundary in action: **integrity-critical logic (Merkle hashing) is locked behind the full package's import path.**

## Practical Consequence

If you're writing new code or integrating, **always import from `reasons`, never from `reasons_lib`**. The shim will eventually be removed (the docstrings already tell you to). The only reason it exists today is so that the `0.58.0` release doesn't break a `pip install ftl-reasons` + `import reasons_lib` in some downstream script.

## Topics to Explore

- [file] `reasons/network.py` — The main network engine (≥516 lines) that calls into `merkle.py`; understanding how `compute_content_hash` and `compute_text_hash` are invoked during node insertion reveals where integrity checking is wired in
- [function] `reasons/merkle.py:verify_node` — The verification entry point (line ~80+); reading its full return contract shows what a "text mutation" finding looks like and how `api.py:verify_all` aggregates them
- [file] `reasons/api.py` — The 2700+ line API layer that ties together network, merkle, metadata, and storage; the `verify_all` and `backfill_hashes` imports at lines 2736/2750 are the integration seam
- [diff] `reasons_lib/` vs. `reasons/` (git history) — Tracing when `reasons_lib` was introduced and when sub-modules were removed from it would reveal the exact rename commit and whether any functionality was dropped during the migration
- [general] `optional-dependencies-strategy` — The `pyproject.toml` extras (`pg`, `cluster`, `mcp`, `llm`, etc.) show how the "full" package is actually a core + plugin architecture; understanding which extras gate which sub-modules clarifies the real distribution boundary

## Beliefs

- `reasons_lib-replaces-nothing` — `reasons_lib/` contains zero implementation code; both of its files (6 lines total) are pure re-exports from `reasons/`, making it a drop-in alias, not an alternative
- `reasons_lib-lacks-merkle` — The `reasons_lib` package does not include a `merkle` sub-module; any `from reasons_lib.merkle import …` raises `ModuleNotFoundError`, whereas `from reasons.merkle import …` succeeds
- `reasons-contains-all-submodules` — Every functional sub-module (`merkle`, `network`, `api`, `cli`, `mcp_server`) lives exclusively under `reasons/`; `reasons_lib/` only re-exports the top-level dataclasses and `metadata`
- `pyproject-ships-both-packages` — The `[tool.setuptools] packages` list in `pyproject.toml` explicitly includes both `"reasons"` and `"reasons_lib"`, so `pip install ftl-reasons` installs the shim as a first-party package alongside the full package
- `merkle-hashing-is-internal` — All 17 codebase references to "merkle" are within `reasons/` internals (`network.py`, `api.py`, `merkle.py`) and `tests/`; no external-facing API or `reasons_lib` shim exposes Merkle functions directly
