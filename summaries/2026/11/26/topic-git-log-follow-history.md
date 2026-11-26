---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Run `git log --follow --name-status -- reasons_lib/__init__.py` to see whether the shim was added directly or renamed from a fuller implementation.

# `git log --follow --name-status -- reasons_lib/__init__.py`

## What the command tells you

`git log --follow --name-status -- <path>` walks a single file's history backwards and, crucially, **tries to follow it across renames**. The `--name-status` flag prefixes each commit with `A` (added), `M` (modified), `R###` (renamed, with similarity score), or `D` (deleted). That means:

- An `A reasons_lib/__init__.py` entry with no preceding lineage → the shim was **created fresh at that path**.
- An `R0xx reasons/__init__.py → reasons_lib/__init__.py` (or similar) → the file was **renamed in from a fuller implementation**.
- A chain of `M` entries on the same path, with earlier content that matches the full implementation → the file was **edited in place** from full implementation to shim.

## What the observations already establish

From `read_reasons_lib_init` and `read_reasons_init`:

- `reasons_lib/__init__.py` is **3 lines**: a docstring ("Backwards-compatibility shim — use `import reasons` instead.") and `from reasons import *  # noqa: F401,F403`.
- `reasons/__init__.py` is **59 lines** of real code (`Justification`, `Node`, `Nogood` dataclasses, lines 1–59).

That is a **3-line vs 59-line asymmetry**. Git's default rename similarity threshold is 50%, so a pure rename between these two contents would *not* be detected. Any `R` entry in the log would therefore have to represent the **opposite** direction: a fuller file renamed *away* (e.g., `reasons_lib/__init__.py` → `reasons/__init__.py`), with the shim subsequently re-added.

Supporting evidence:

- `pyproject.toml` ships **both** packages: `packages = ["reasons", "reasons_lib"]`, so both live side-by-side in the same tree — not one path replacing the other.
- `CLAUDE.md` line 24: "`reasons/` — importable as `import reasons` (also aliased as `reasons_lib` for backwards compat)" and line 35: "`reasons_lib/` — backwards-compat shim — re-exports from `reasons/`".
- `CHANGELOG.md` line 136 references a **past** `reasons_lib.api` that returned dicts for CLI and tool-call consumers — evidence that `reasons_lib` once held real implementation, before today's shim.
- Every module in `reasons_lib/` (`api.py`, `ask.py`, `cli.py`, `cluster.py`, …) is the same 2-line "Backwards-compatibility shim — use `from reasons.<x>` instead." pattern (`find_reasons_package_forwarding_imports`, total_count 283), confirming the whole directory is a compat layer, not a partial one.
- `find_reasons_lib_references_in_python` returns **0 matches** — nothing in the codebase imports `reasons_lib` today, so it is dead-but-public surface kept only for external backwards compat.

## Predicted output and how to read it

Expect one of two shapes:

1. **`A reasons_lib/__init__.py`** (and later `M` entries on the same path) → the shim was **added directly** as a new file. The 59-line full implementation in `reasons/__init__.py` has its own, separate lineage (you'd have to run `git log --follow -- reasons/__init__.py` to find where *that* full implementation originally came from — likely a rename from `reasons_lib/` or from `rms/`, per `CHANGELOG.md` line 109 "Renamed from `rms` to `reasons`").

2. **`M reasons_lib/__init__.py` preceded by an older `A reasons_lib/__init__.py`** whose blob at that commit contains the `Justification`/`Node`/`Nogood` classes → the file was **edited in place** in a single commit that replaced the full implementation with the shim (a "gut the module" refactor). In that case `--follow` will surface the pre-shim content naturally.

What you will **not** see (and this is the useful diagnostic):

- No `R` entry pointing from `reasons/__init__.py` into `reasons_lib/__init__.py`, because a 3-line wildcard import is too dissimilar to a 59-line dataclass definition to cross the 50% similarity threshold. If `--follow` reports a rename at all, it is the *outbound* direction (full implementation leaving `reasons_lib/`).

## Important caveat

The actual stdout of `git log --follow --name-status -- reasons_lib/__init__.py` is **not present** in the provided observations. The prediction above is derived from the current file contents (`read_reasons_lib_init`, `read_reasons_init`), packaging (`pyproject.toml`), documentation (`CLAUDE.md`, `CHANGELOG.md`), and the fact that no Python file references `reasons_lib` (`find_reasons_lib_references_in_python`, total_count 0). To answer definitively, run the command and check whether the oldest entry on the shim's path is `A` (added directly) or whether earlier `M`/`R` entries reveal a fuller implementation.

## Topics to Explore

- [file] `reasons/__init__.py` — Confirm the full `Justification`/`Node`/`Nogood` implementation lives here; run `git log --follow` on this path to see whether it was renamed from `reasons_lib/` or `rms/` in the past
- [file] `reasons_lib/api.py` — Second example of the same shim pattern; useful cross-check that the entire `reasons_lib/` tree was converted, not just `__init__.py`
- [file] `CHANGELOG.md` — Lines 109 and 136 document both the `rms → reasons` rename and the earlier existence of a real `reasons_lib.api`, which is the historical evidence `--follow` should corroborate
- [diff] `git log --follow -- reasons/__init__.py` — The complementary query: reveals whether the *target* package was renamed in from a fuller `reasons_lib` implementation (which would make the shim's `A` entry the intentional "re-add")
- [general] `backwards-compat-shim-policy` — How long `reasons_lib` is intended to live, and whether it is tracked/released or slated for removal in the `ftl-reasons` package (`pyproject.toml` still lists it in `packages`)

## Beliefs

- `reasons-lib-is-a-shim` — `reasons_lib/__init__.py` is a 3-line wildcard re-export of `reasons`, not an independent implementation.
- `reasons-lib-requires-reasons` — Importing `reasons_lib` depends entirely on `reasons` being importable, because every module is a `from reasons.<x> import *`.
- `reasons-holds-full-implementation` — The 59-line `Justification`/`Node`/`Nogood` definitions live in `reasons/__init__.py`, not `reasons_lib/__init__.py`.
- `reasons-lib-is-packaged-alongside-reasons` — `pyproject.toml` declares `packages = ["reasons", "reasons_lib"]`, so both are shipped from the `ftl-reasons` distribution.
- `reasons-lib-has-no-internal-importers` — The codebase has zero Python references to `reasons_lib` (search returned 0 matches); it exists solely for external backwards compatibility.
