---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/cli.py

# `reasons_lib/cli.py`

```python
"""Backwards-compatibility shim — use ``from reasons.cli`` instead."""
from reasons.cli import *  # noqa: F401,F403
```

## Purpose

This module owns **no logic at all**. It is a two-line re-export shim whose only job is to keep the legacy import path `reasons_lib.cli` working while the canonical implementation lives in `reasons/cli.py`.

The repository ships two parallel package trees (`reasons/` and `reasons_lib/`) with identical module names. `reasons_lib` is the older/user-facing namespace; `reasons` is the current one. This file is the migration seam for the CLI module specifically — the `reasons_lib` versions of `api.py`, `ask.py`, `storage.py`, etc. follow the same pattern.

Its contract to callers is: *"anything importable from `reasons.cli` is importable from `reasons_lib.cli`."*

## Key Components

There are none of its own — no classes, functions, constants, or `__all__`:

- **Module docstring** (line 1) — the only documentation; it explicitly names `reasons.cli` as the replacement and marks this file as deprecated-by-convention.
- **`from reasons.cli import *`** (line 2) — the entire implementation. It copies every public name from `reasons.cli` into this module's namespace.
- **`# noqa: F401,F403`** — suppresses `pyflakes`/`ruff` on both counts: `F401` (imported but unused — true, we only re-export) and `F403` (star import used — true, intentional).

The effective public surface is therefore **whatever `reasons.cli` declares**. If `reasons/cli.py` defines `__all__`, that list is authoritative; if it does not, Python's default applies (every module-global not starting with `_`). That includes `main`, the `argparse` parser builder, and any command handler functions in the real CLI.

## Patterns

1. **Compatibility shim / module aliasing** — the classic "old path forwards to new path" pattern. Chosen over `sys.modules['reasons_lib.cli'] = reasons.cli` (which would make them literally the same module object) and over `from reasons.cli import main, ...` (which would require maintenance on every new CLI symbol).
2. **Star-import as a proxy** — trades explicit `__all__` curation for automatic drift-following. New commands added to `reasons/cli.py` appear here for free.
3. **Dual-package layout** — a convention spanning the whole repo: `reasons_lib/<module>.py` mirrors `reasons/<module>.py`. Consistency across the tree is what makes the shim auditable.

## Dependencies

**Depends on:** exactly one thing — `reasons.cli`. There is no `import argparse`, no `sys`, no third-party import. This is deliberate: the shim adds zero import cost beyond what `reasons.cli` already pays.

**Depended on by:** a wide set of consumers, most importantly:

- `reasons/cli.py` — ⚠️ listed as importing this shim, which would be *circular* (`reasons.cli` → `reasons_lib.cli` → `reasons.cli`). In practice this is almost certainly a guarded fallback (`try: ... except ImportError:`) or an in-function import; if it is a module-level unguarded import, the two modules only resolve because `reasons.cli` is fully initialized before `reasons_lib.cli` is first imported. **Verify this before touching either file.**
- `reasons/mcp_client.py` and `reasons_lib/mcp_client.py` — the MCP client reuses CLI helpers (e.g. for configuration/argument handling) via the legacy path.
- The test suite: `tests/test_cli.py`, `tests/test_ask.py`, `tests/test_ask_mcp.py`, `tests/test_build_wiki.py`, `tests/test_contradictions.py`, `tests/test_derive.py`, `tests/test_example.py`, `tests/test_hf_default_org.py`, `tests/test_pg_dispatch.py`, `tests/test_repair.py`, `tests/test_repair_premises.py`, `tests/test_report_belief.py`, `tests/test_report_gated.py`, `tests/test_research.py`, `tests/test_review.py`, `tests/test_review_premises.py`, `tests/test_roundtrip.py`, `tests/test_search_sources.py`, `tests/test_source_type.py`, `tests/test_topics.py`, `tests/test_verify.py`.

That breadth of test usage is the real constraint here: the shim is load-bearing for ~20 test modules, so removing it requires a mechanical rewrite of those imports, not just deleting a file.

## Flow

Trivial, but the timing matters:

1. An importer executes `import reasons_lib.cli` (or `from reasons_lib.cli import main`).
2. Python loads `reasons_lib/cli.py`, running line 2, which triggers a **full, eager load of `reasons.cli`** — and `reasons.cli` in turn pulls in whatever it needs (argparse setup, storage, API, etc.).
3. `import *` copies the resolved public names into `reasons_lib.cli.__dict__`. This is a **snapshot at first-import time**: a plain `dict` binding copy, not a lazy proxy.
4. Downstream code calls the copied objects. Note these are the *same function objects* as in `reasons.cli` — calling `reasons_lib.cli.main(...)` runs exactly `reasons.cli.main(...)`, with `__module__ == "reasons.cli"`.

There is no data transformation, no dispatch, no state.

## Invariants

- **No definitions allowed.** Anything defined locally in this file would shadow the canonical symbol and silently diverge. Keep it a pure re-export.
- **Single source of truth.** `reasons.cli` must never import CLI symbols *from* `reasons_lib.cli` at module scope; the dependency direction is one-way (`reasons_lib` → `reasons`).
- **Public-only forwarding.** Private names (`_foo`) and undeclared submodules are **not** re-exported, because `import *` skips underscore-prefixed names and only includes submodules if `reasons.cli.__all__` lists them. So `from reasons_lib.cli import _build_parser` or `reasons_lib.cli.argparse` will `ImportError`/`AttributeError` even though they work against `reasons.cli`.
- **Eager import.** The shim cannot be imported if `reasons.cli` fails to import. There is no lazy indirection.
- **Distinct namespaces.** `reasons_lib.cli` and `reasons.cli` are two separate module objects with two separate `__dict__`s that happen to point at the same objects initially.

## Error Handling

**None.** The file has no `try`/`except`, no `DeprecationWarning`, no logging.

Consequences a maintainer must know:

- Any `ImportError`/`SyntaxError`/side-effect failure inside `reasons/cli.py` propagates verbatim when someone imports `reasons_lib.cli`. The traceback will point at `reasons/cli.py`, which is correct but can confuse someone who only edited the `reasons_lib` tree.
- Failures are **not** soft-failed into a stub — `from reasons_lib.cli import main` raises rather than returning `None`.
- Deprecation is communicated only by the docstring; nothing is emitted at runtime, so there is no telemetry telling you who still uses the legacy path.
- **Monkeypatching caveat (the biggest practical gotcha):** because names are copied, `monkeypatch.setattr("reasons_lib.cli.some_helper", fake)` rebinds the *shim's* attribute only. Internal calls inside `reasons/cli.py` resolve `some_helper` from `reasons.cli.__dict__` and will **not** see the patch. Tests that patch through the legacy path can therefore appear to work while exercising the real implementation — or vice versa. Always patch `reasons.cli.<name>`.

## Topics to Explore

- [file] `reasons/cli.py` — The real implementation this file forwards to; check whether it defines `__all__` (which would define this shim's exact surface) and whether it imports from `reasons_lib.cli` at module scope (circularity risk).
- [file] `pyproject.toml` — Confirms whether both `reasons` and `reasons_lib` are packaged and which module path the console-script entry point (`[project.scripts]`) targets.
- [function] `reasons/cli.py:main` — The actual command dispatcher and `argparse` setup that the star import exposes through the legacy path.
- [general] `dual-package-migration` — Survey how many `reasons_lib/*.py` files are pure shims versus real implementations; that ratio determines whether `reasons_lib` can be deleted wholesale or needs a staged deprecation.
- [file] `tests/test_cli.py` — Shows which symbols tests expect from the CLI module and whether any of them patch through the legacy path (see the monkeypatch caveat above).

## Beliefs

- `reasons-lib-cli-is-a-shim` — `reasons_lib/cli.py` contains exactly one executable statement and defines no functions, classes, or constants of its own.
- `shim-reexport-surface-maps-to-reasons-cli-all` — The names available from `reasons_lib.cli` are exactly `reasons.cli.__all__` when defined, otherwise every non-underscore module-global of `reasons.cli`.
- `shim-creates-independent-name-bindings` — `reasons_lib.cli` and `reasons.cli` are distinct module objects sharing the same initial object references, so rebinding an attribute on one does not affect the other.
- `shim-import-is-eager-and-unprotected` — Importing `reasons_lib.cli` fully imports `reasons.cli` and propagates any of its import-time exceptions, with no try/except in the shim.
- `shim-does-not-forward-private-names-or-submodules` — Star-import semantics prevent `reasons_lib.cli` from exposing underscore-prefixed names or submodules such as `reasons.cli.argparse`.
- `reasons-lib-cli-imported-by-reasons-cli-raises-circularity-question` — The dependency listing claims `reasons/cli.py` imports this shim, which is only safe if that import is guarded or deferred.
