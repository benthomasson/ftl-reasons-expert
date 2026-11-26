---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Useful pattern for enforcing codebase invariants between two distant declarations without running the CLI.

## The pattern (and a caveat)

Short version: **turn each side of the invariant into an importable object (a dict, a list, an `__all__`, an argparse parser), then in a pytest test import both sides and assert they agree — or better, collapse the two declarations into one so the invariant holds by construction and can never drift.**

Two important caveats about *this* repo, based on the observations:

1. **This pattern is not actually implemented.** `search_cli_in_tests` returned `total_count: 0` for `CLI|cli|without running`, and `find_candidate_tests_for_cross_declaration_checks` returned `total_count: 0` for `assert.*(__all__|COMMANDS|KEYS|DEFAULT|VERSION|ROUTES|registry|choices)`. The test suite described in `CLAUDE.md` (211 tests across `test_network.py`, `test_api.py`, `test_merkle.py`, etc.) covers the propagation engine, not cross-declaration parity.
2. **The observations don't reveal enough of `reasons/cli.py`'s structure** to say which variant of the pattern is best here. We see `cmd_*` handlers (lines 19–160+) and one example `p.add_argument(...)` at line 3438, but not how the parser is built (inline in `main()` vs. a `build_parser()` helper) or how a parsed command reaches a handler (`if/elif` chain vs. dispatch dict). That's the piece that determines whether you can *eliminate* the duplication or only *test* it. Answer below says what the pattern looks like in each case, using the couples that the observations *do* show.

---

## Where the distant declarations live in this codebase

### 1. Shim parity — `reasons_lib/*` vs `reasons/*` (the good case)

The observations show 283+ matches for `reasons_lib|from reasons|import reasons`, and every `reasons_lib/*.py` file shown follows the same 3-line shape:

```python
# reasons_lib/cli.py:1-2
"""Backwards-compatibility shim — use ``from reasons.cli`` instead."""
from reasons.cli import *  # noqa: F401,F403
```

This is exactly the right shape for a version of the pattern: **the shim declares nothing of its own**, so there is nothing to drift. When `reasons/cli.py` gains `cmd_foo`, `reasons_lib/cli.py` gains it too — the invariant is enforced by *construction*, not by a test. Same in `reasons_lib/__init__.py:2` (`from reasons import *`) and `reasons_lib/api.py:2`.

The only residual risk is the opposite direction: a shim that adds a name the source doesn't have. That's a one-line test, and it never invokes the CLI:

```python
# tests/test_shim_parity.py (proposed — not present today)
import importlib

def test_shim_public_api_superset_of_source():
    for name in ["cli", "api", "ask", "cluster", "compact", "derive", "hf"]:
        src = importlib.import_module(f"reasons.{name}")
        shim = importlib.import_module(f"reasons_lib.{name}")
        public = lambda m: {n for n in vars(m) if not n.startswith("_")}
        assert public(shim) >= public(src), f"{name} shim missing names"
```

This is a genuine "two distant declarations" invariant — the module list in `reasons/` vs. the shim list in `reasons_lib/` — and it stays green without ever running `reasons`.

### 2. Argparse subparsers vs. `cmd_*` dispatch (the risky case)

`reasons/cli.py` contains both halves of the same fact — "these are the commands the CLI supports":

- **Declaration A**: `p.add_argument(...)` on subparsers, e.g. `reasons/cli.py:3438` (`p.add_argument("agent_name", help="Agent name (must match previous import)")`). Notice the help text is itself a hand-written cross-declaration warning — a symptom that the invariant is not tested.
- **Declaration B**: the `cmd_*` functions at `reasons/cli.py:19, 44, 67, 85, 160` (`cmd_init`, `cmd_add`, `cmd_add_justification`, `cmd_remove_justification`, `cmd_mark_duplicate`).

If you add `reasons add-foo` to argparse but forget `cmd_add_foo`, nothing fails until a user types the command. Two ways to close the gap, neither of which spawns `subprocess.run(["reasons", ...])`:

**Eliminate the duplication (preferred).** One registry drives both registration and dispatch:

```python
COMMANDS = {
    "add": cmd_add,
    "add-justification": cmd_add_justification,
    "retract": cmd_retract,
    "mark-duplicate": cmd_mark_duplicate,
    ...
}

def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser()
    sub = parser.add_subparsers(dest="command", required=True)
    for name, handler in COMMANDS.items():
        sp = sub.add_parser(name)
        register_args(sp, name)          # per-command flags only
        sp.set_defaults(func=handler)
    return parser
```

Now there is only one declaration, and the invariant is enforced by construction. `build_parser()` is trivially unit-testable:

```python
def test_every_subcommand_has_a_handler():
    from reasons.cli import COMMANDS, build_parser
    sub = next(a for a in build_parser()._actions
               if isinstance(a, argparse._SubParsersAction))
    assert set(sub.choices) == set(COMMANDS)
```

**Or assert on the two declarations.** If you cannot refactor immediately, still test the invariant instead of running the CLI:

```python
def test_all_registered_subcommands_have_cmd_handlers():
    import argparse, reasons.cli as cli
    sub = next(a for a in cli.main_parser()._actions   # requires build_parser()
               if isinstance(a, argparse._SubParsersAction))
    expected = {"cmd_" + name.replace("-", "_") for name in sub.choices}
    actual = {n for n in vars(cli) if n.startswith("cmd_")}
    assert expected <= actual, expected - actual
```

This never parses `sys.argv`; it only introspects the parser object.

### 3. Version parity — `pyproject.toml` vs `reasons/cli.py:9`

`reasons/cli.py:9` imports `from importlib.metadata import version as _pkg_version`, so the printed version originates from package metadata, not a literal. That means the two declarations (`pyproject.toml`'s `project.version` and whatever `importlib.metadata` resolves) already agree *by construction*. A test that reads `pyproject.toml` via `tomllib` and compares to `_pkg_version("reasons")` locks that in:

```python
def test_version_matches_pyproject():
    import tomllib
    from importlib.metadata import version
    with open("pyproject.toml", "rb") as f:
        declared = tomllib.load(f)["project"]["version"]
    assert version("reasons") == declared
```

### 4. Cross-module private imports — `reasons/pg.py:943, 1324`

```python
from reasons.api import _STOP_WORDS           # reasons/pg.py:943
from reasons.api import _TOPIC_STOP_WORDS     # reasons/pg.py:1324
```

Importing a leading-underscore name across modules is already a *runtime* invariant (rename it and `pg.py` fails to import), but nothing asserts the *shape* (a `set`/`frozenset`, not `None`, non-empty). A one-liner in `tests/test_api.py` would be enough, still no CLI involved.

---

## Why "without running the CLI" matters here

- Every `cmd_*` function in `reasons/cli.py` starts with `api.<something>(...)` (lines 19–160), which means invoking the CLI hits SQLite (per `CLAUDE.md`) and possibly `REASONS_PG_CONNINFO` (`reasons/cli.py:161`). Slow and environment-sensitive.
- The parser is a plain Python object — `parser._actions`, `parser._subparsers`, `sub.choices` are all introspectable. You never need `reasons --help`.
- The parity you actually care about (source module ↔ shim, subcommand ↔ handler, version ↔ metadata) is expressed entirely in imported names, so `vars(module)`, `inspect.getmembers`, or a tiny `ast` walk over `reasons/cli.py` gives you the same signal at a fraction of the cost.

---

## What's missing from the observations

- **How the parser is built.** Only one `p.add_argument(...)` snippet (`reasons/cli.py:3438`) is shown; we can't tell whether it's in `main()` (hard to test) or in a `build_parser()` helper (easy to test). This determines whether the "eliminate the duplication" path is a small refactor or a big one.
- **How a parsed command dispatches.** If `main()` is a giant `if command == "add": ... elif ...`, that's the exact line to replace with a `COMMANDS` dict. If it already uses a dispatch table, parity may just need a test.
- **Whether `reasons/cli.py` defines `__all__`.** If it does, `import *` in `reasons_lib/cli.py` is precise; if not, the shim also re-exports any public helper that ever gets added to `reasons/cli.py`, which may not be intended.
- **The test directory layout.** `CLAUDE.md` lists the test files but the observations don't include a `tests/conftest.py` or a sample test, so the idioms used in the suite (pytest vs. unittest, fixtures vs. raw asserts) are unknown.

With those four things visible, this would become a concrete "rename the dict and delete the `if/elif`" plan instead of a sketch.

## Topics to Explore

- [function] `reasons/cli.py:main` — Where subparser registration and command dispatch meet; look for a `if command == ...` chain that a `COMMANDS` dict would replace.
- [file] `reasons/cli.py` — The 4015-line file containing both the `cmd_*` handlers (lines 19–160+) and the `p.add_argument` calls (e.g., line 3438); check whether a `build_parser()` helper exists.
- [file] `reasons_lib/cli.py` — The 3-line wildcard shim; check whether `reasons.cli` defines `__all__`, which determines exactly what the shim re-exports.
- [general] `shim-parity-test` — Whether a test asserting `public(reasons_lib.X) >= public(reasons.X)` for every shim would be a cheap, no-CLI way to guard the 283 `reasons_lib` references.
- [general] `version-source-of-truth` — How `pyproject.toml`'s version reaches `_pkg_version("reasons")` at `reasons/cli.py:9`, and whether a lockstep test is worth writing.

## Beliefs

- `reasons-lib-is-a-thin-shim` — Every module in `reasons_lib/` is a 3-line file whose only body is `from reasons.<name> import * # noqa: F401,F403`.
- `reasons-lib-cli-inherits-public-surface` — `reasons_lib/cli.py` declares no names of its own, so its public surface equals `reasons/cli.py`'s public surface, modulo `__all__`.
- `no-cross-declaration-invariant-tests-exist` — Searches for `assert.*(__all__|COMMANDS|KEYS|DEFAULT|VERSION|ROUTES|registry|choices)` and `CLI|cli|without running` in `tests/` returned `total_count: 0`.
- `cli-subcommands-require-cmd-handlers` — Argparse subparsers registered in `reasons/cli.py` (e.g., line 3438) must have matching `cmd_*` functions (lines 19–160+), but nothing in the test suite enforces this.
- `pg-depends-on-api-stop-words` — `reasons/pg.py:943` imports `_STOP_WORDS` and `reasons/pg.py:1324` imports `_TOPIC_STOP_WORDS` from `reasons.api`, creating a private-name coupling between two modules.
