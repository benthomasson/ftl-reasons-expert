---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Determines whether `reasons_lib/` re-exports from `reasons/` or contains duplicated mirrored modules.

## Answer: `reasons_lib/` is a pure re-export shim, not a mirrored copy

Every file sampled in `reasons_lib/` is a 3-line "backwards-compatibility shim" whose entire body is a star re-export from the corresponding `reasons/` module. There is no duplicated logic, no parallel dataclasses, and no shadow implementations.

### Evidence from the observations

| File | Lines | Body signature | Re-exports from |
|---|---|---|---|
| `reasons_lib/__init__.py` | 3 | `from reasons import *` (line 2) | `reasons` |
| `reasons_lib/api.py` | 3 | `from reasons.api import *` (line 2) | `reasons.api` |
| `reasons_lib/cli.py` | 3 | `from reasons.cli import *` (line 2) | `reasons.cli` |

Each shim's docstring even names the canonical target, e.g. `reasons_lib/api.py:1` reads:
> `"""Backwards-compatibility shim — use ``from reasons.api`` instead."""`

### The canonical source lives in `reasons/`

- `reasons/__init__.py` (59 lines) actually **defines** the public data model — `Justification`, `Node`, and `Nogood` dataclasses — with docstrings, fields, and `field(default_factory=...)` defaults.
- `reasons/api.py` is 5886 lines of real implementation (the functional API: `_is_visible`, `_filter_by_tags`, `_resolve_namespace`, `_with_network`, etc.).
- The `from_imports` parsed from `reasons_lib/api.py` confirms a single edge: `{"module": "reasons.api", "names": ["*"]}` — no other imports, no reimplementation.

A "duplicated mirrored module" would show up as a `reasons_lib/api.py` containing hundreds/thousands of lines with its own definitions. Instead it is 3 lines. Same for `__init__.py` (3 vs 59) and `cli.py` (3 lines). The line-count ratio (~0.05%) is the smoking gun: these are shims, not mirrors.

### One inconsistency worth flagging

`grep_reasons_lib_for_reexports` reports **0 matches** for `from reasons|import reasons`, yet `reasons_lib/__init__.py:2` literally contains `from reasons import *`. Either that grep was scoped to a subset of files (e.g., only `reasons/`), ran before the shim existed, or used a pattern that didn't recurse. The **file reads are authoritative** here, and they agree unanimously on the shim pattern. If you ever rely on that grep in CI (e.g., to enforce "no backward imports"), verify its scope first — it's currently blind to the very thing it was supposed to find.

### What this means practically

- `import reasons_lib` and `from reasons_lib.api import foo` will resolve to the same objects as `reasons` / `reasons.api`, so identity checks (`reasons.Justification is reasons_lib.Justification`) should pass.
- Any `isinstance` / dataclass equality checks are safe — there is no risk of two `Node` classes drifting apart.
- `from x import *` is governed by `__all__` (or, absent it, all non-underscore names). If `reasons/` ever adds a public symbol but forgets `__all__`, the shim will leak underscore-prefixed names. Explore that next.

### What's missing from the observations

- No read of `reasons_lib/network.py`, `reasons_lib/storage.py`, `reasons_lib/metadata.py`, or `reasons_lib/pubsub.py` — the shim pattern is inferred from three samples. It's almost certainly uniform, but if a fourth file is a real module, this conclusion would need revising.
- No `__all__` definitions shown — so we can't confirm exactly which names the star-import forwards.
- No check of whether `reasons/` itself imports from `reasons_lib/` anywhere (would create a cycle / shadowing).

## Topics to Explore

- [file] `reasons_lib/network.py` — Confirm the remaining shim modules follow the same 3-line pattern and no stray real code lives under `reasons_lib/`
- [file] `reasons/__init__.py` — Check whether `__all__` is declared; it governs exactly what `from reasons import *` forwards through the shim
- [function] `reasons/api.py:_with_network` — The load/cache/save context manager backing the public API; the shim's correctness ultimately depends on this being the one true implementation
- [general] `backwards-compat-deprecation-plan` — Understand why these shims exist (renaming `reasons_lib` → `reasons`) and when they can be removed
- [general] `grep-scope-verification` — Re-run the `from reasons` grep recursively over `reasons_lib/` to confirm the reported 0 matches is a tooling artifact rather than a real absence

## Beliefs

- `reasons-lib-is-backcompat-shim` — Every sampled `reasons_lib/*.py` file is a 3-line backwards-compatibility shim, not a duplicated implementation.
- `reasons-lib-reexports-via-star-import` — `reasons_lib/__init__.py`, `reasons_lib/api.py`, and `reasons_lib/cli.py` each contain exactly one `from reasons.<sub> import *` on line 2.
- `reasons-is-canonical-source` — `reasons/__init__.py` (59 lines) and `reasons/api.py` (5886 lines) contain the actual implementations of `Justification`, `Node`, `Nogood`, and the functional API.
- `reasons-lib-has-no-additional-imports` — Parsed imports for `reasons_lib/api.py` show only `{"module": "reasons.api", "names": ["*"]}`, proving no extra dependencies or local code.
- `reasons-lib-shim-preserves-object-identity` — Because the shims use `import *` from the canonical modules, symbols imported via `reasons_lib` are the same objects as those in `reasons`.
