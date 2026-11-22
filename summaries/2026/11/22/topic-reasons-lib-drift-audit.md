---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: `diff -r reasons reasons_lib` to quantify divergence; the missing `merkle.py` is already one confirmed gap.

# `diff -r reasons reasons_lib` — Quantifying Divergence

## What the command actually tells you

`diff -r` walks both directory trees and reports three kinds of lines:

1. `Only in reasons: <file>` — modules present in the real package but absent from the shim.
2. `Only in reasons_lib: <file>` — stale leftovers that shouldn't exist (expected: none).
3. `Files reasons/<x> and reasons_lib/<x> differ` — for the files that exist in both.

Given observations, the diff is lopsided. `reasons/` is a real package (`reasons/__init__.py` is 59 lines defining `Justification`, `Node`, `Nogood`), while `reasons_lib/__init__.py` is a 3‑line trampoline:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
from reasons import *  # noqa: F401,F403
```

So the only file that exists on both sides is `__init__.py`, and it *"differs"* in a way that's total, not partial — one is a dataclass module, the other is a re-export. Every other module in `reasons/` (including `merkle.py`, plus `api.py`, `storage.py`, `network.py`, `cli.py`, `mcp_server.py`, etc., referenced via `find_merkle_references` and `pyproject.toml`) shows up as `Only in reasons: …`.

## The confirmed `merkle.py` gap

`reasons/merkle.py` is a real 147-line module exporting `compute_text_hash`, `compute_merkle_hash`, `compute_content_hash`, `verify_node`, `verify_justification`, `verify_all`, and `backfill_hashes`. It's imported (per `find_merkle_references`) from:

- `reasons/api.py:2736` — `from .merkle import verify_all`
- `reasons/api.py:2750` — `from .merkle import backfill_hashes as _backfill`
- `reasons/network.py:130, 514, 516` — `compute_text_hash` / `compute_content_hash`
- `tests/test_merkle.py:7` — the full import surface

`reasons_lib/merkle.py` does not exist, and — critically — the `from reasons import *` in `reasons_lib/__init__.py` won't help. `reasons/__init__.py` never imports the `merkle` submodule into its own namespace, so `reasons_lib.merkle` is simply not bound. That gap is the "confirmed" one because it's the one an external caller would hit first: `from reasons_lib.merkle import compute_text_hash` raises `ModuleNotFoundError` today, whereas `from reasons.merkle import compute_text_hash` works.

`merkle.py` is a symptom, not a special case. Every top-level submodule of `reasons/` is missing from `reasons_lib/` for the same structural reason: the shim only forwards the *names defined or imported inside* `reasons/__init__.py`, not the submodule tree.

## Quantifying the divergence properly

To turn the raw `diff -r` output into a number, count the "Only in reasons:" lines and compare to the number of modules listed in `reasons/__init__.py`'s sibling tree. Based on what we see:

| Metric | Value | Source |
|---|---|---|
| Files in `reasons/` | many (at least `__init__.py`, `merkle.py`, `api.py`, `storage.py`, `network.py`, `cli.py`, `mcp_server.py`) | `find_merkle_references`, `pyproject.toml` scripts |
| Files in `reasons_lib/` | 1 (`__init__.py`) inferred | 3-line `__init__.py`, no other submodule seen |
| Files present in both | 1 (`__init__.py`) | `compare_reasons_exports`, `compare_reasons_lib_exports` |
| Files differing | 1 (`__init__.py`) | contents above |
| Files missing on `reasons_lib` side | all submodules (incl. `merkle.py`) | `diff -r` output, one gap already confirmed |

So the *ratio* of missing modules is roughly 100% of the submodule tree. The only thing the shim actually reproduces is the top-level namespace: `reasons_lib.Justification`, `reasons_lib.Node`, `reasons_lib.Nogood` (plus incidental leakage like `reasons_lib.dataclass` and `reasons_lib.field`, since `reasons/__init__.py` has no `__all__` to suppress them).

## Is the divergence a problem?

Two pieces of evidence cut in opposite directions:

- **Packaging says yes.** `pyproject.toml` has `packages = ["reasons", "reasons_lib"]`, so `reasons_lib` ships to every wheel user. Keeping a published package whose only file is a shim that fails for `reasons_lib.merkle` (and `reasons_lib.api`, `reasons_lib.storage`, …) is misleading at best. `CHANGELOG.md:136` even records historical usage of `reasons_lib.api` — that import path already exists in the wild.
- **Grep says maybe not.** `find_reasons_lib_references` returns `total_count: 0` — no in-repo code imports `reasons_lib` at all. The shim is retained purely for external callers, and nothing in the repo's own test suite exercises it. `CLAUDE.md:24` and `CLAUDE.md:35` describe it as a shim, so the intent is documented but not enforced.

The practical answer is that `diff -r reasons reasons_lib` is not measuring "which files drifted" — it's measuring "the shim was never intended to be a mirror, but nothing prevents downstream code from expecting it to be one." `merkle.py` is just the module that finally surfaced the expectation.

## What the diff cannot tell you

- Whether any *out-of-tree* consumers actually do `from reasons_lib.merkle import …` — the evidence only covers this repo.
- Whether a previous version of `reasons_lib/` contained real copies (the CHANGELOG entry hints yes, but snapshot history isn't in the observations).
- Whether `reasons_lib` should be deleted outright or converted into a fanout of `from reasons.X import *` shims. That's a design call, not a diff result.

## Topics to Explore

- [diff] `reasons vs reasons_lib` — Re-run `diff -r reasons reasons_lib` and record the exact `Only in reasons:` list; this is the ground truth for how many modules (beyond `merkle.py`) the shim fails to expose.
- [file] `pyproject.toml` — Confirm what `packages = ["reasons", "reasons_lib"]` ships, and whether dropping `reasons_lib` is acceptable for the `0.58.0` → next version bump.
- [function] `reasons/api.py:check_integrity` — The user-facing entry point that calls `verify_all`; verify it stays on `reasons.merkle` and never routes through `reasons_lib`.
- [general] `backwards-compat-shim-policy` — Decide whether `reasons_lib` should be (a) deleted, (b) a full `from reasons.X import *` fanout including `merkle`, `api`, `storage`, `network`, or (c) documented as names-only (equivalent to `from reasons import *` today).
- [file] `CHANGELOG.md` — Line 136 references `reasons_lib.api`; trace when that path was last useable to decide whether the shim is a live contract or legacy cruft.

## Beliefs

- `reasons-lib-is-a-shim-not-a-mirror` — `reasons_lib/__init__.py` re-exports from `reasons` via `from reasons import *` and contains no submodules, so `reasons_lib.merkle` is unbound.
- `merkle-gap-is-structural-not-isolated` — The missing `reasons_lib/merkle.py` reflects the same omission that also hides `api`, `storage`, `network`, `cli`, and `mcp_server` from the shim.
- `reasons-merkle-is-imported-only-via-reasons` — Every reference to `merkle` in the repo uses `reasons.merkle` (or relative `.merkle`), never `reasons_lib.merkle`.
- `reasons_lib-has-zero-internal-consumers` — `find_reasons_lib_references` returns 0 matches, so any reliance on the shim originates outside this repository.
- `package-list-includes-both` — `pyproject.toml` `packages` lists both `"reasons"` and `"reasons_lib"`, so the shim is distributed to end users even though the repo doesn't use it.
