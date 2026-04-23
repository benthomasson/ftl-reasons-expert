# File: reasons_lib/cli.py

**Date:** 2026-04-23
**Time:** 16:47

## `reasons_lib/cli.py` — Explanation

### 1. Purpose

This file is the CLI entry point for `ftl-reasons`, a **Reason Maintenance System** (RMS). It owns the terminal interface: argument parsing, human-readable output formatting, and exit codes. It is explicitly a **thin wrapper** — every command delegates to `reasons_lib.api` for business logic and formats the returned dict for the terminal. The one exception is `cmd_propagate`, which drops directly to `Storage` (bypassing `api`).

### 2. Key Components

**`main()`** — the entry point. Builds an `argparse` parser with 30+ subcommands, dispatches via a `commands` dict mapping command names to `cmd_*` functions. Every subcommand gets a `--db` flag defaulting to `api.DEFAULT_DB`.

**Command handlers** (`cmd_add`, `cmd_retract`, `cmd_status`, etc.) — each follows the same contract:
- Accept an `argparse.Namespace` as the sole argument
- Call one `api.*` function, passing through CLI args
- Format the returned dict as human-readable terminal output
- On error, print to stderr and `sys.exit(1)`

**`_derive_one_round(args, round_num=None)`** — the most complex function (~100 lines). Orchestrates a single derivation round: loads the network, builds a prompt, shells out to a model CLI (`claude` or `gemini`) via `asyncio.create_subprocess_exec`, parses the response into proposals, and either writes them to a file for review or auto-applies them. Returns the count of added beliefs (0 = saturated, -1 = error).

**Helper formatters:**
- `_print_cascade(result)` — formats went-out/went-in lists after retraction or assertion
- `_print_restoration_hints(hints)` — suggests `add-justification --any` when multi-premise nodes go OUT with surviving premises
- `_warn_multi_premise(premise_count, any_mode)` — warns at add-time when an SL has 3+ premises and `--any` wasn't used
- `_print_what_if_results(result, action, node_id)` — shared formatting for `what-if retract` and `what-if assert`, including depth-grouped output

### 3. Patterns

**Thin CLI / Fat API**: The module enforces a strict separation. CLI functions never touch the database directly (except `cmd_propagate`). All data flows through `api.*` functions that return plain dicts. This makes the API independently testable and usable from other Python code.

**Uniform error handling idiom**: Every `cmd_*` function wraps the `api` call in `try/except`, catching specific exceptions (`KeyError`, `ValueError`, `FileNotFoundError`), printing to stderr, and exiting with code 1. No exceptions propagate to the caller.

**Dict-based result protocol**: API functions return dicts with well-known keys (`changed`, `truth_value`, `node_id`, etc.). The CLI destructures these for display. This avoids coupling the CLI to domain objects.

**Lazy imports**: `cmd_propagate` imports `Storage` inline; `_derive_one_round` imports `asyncio`, `os`, `shutil`, and the `derive` module inline. This keeps the CLI's startup cost low — heavy dependencies are only loaded for commands that need them.

**Dispatch table**: `main()` maps command strings to functions via a flat dict rather than using `set_defaults(func=...)`, keeping the routing explicit and easy to audit.

### 4. Dependencies

**Imports:**
- `reasons_lib.api` — the primary dependency; nearly every command delegates here
- `reasons_lib.storage.Storage` — used only by `cmd_propagate` (direct DB access)
- `reasons_lib.derive` — used by `_derive_one_round` and `cmd_accept` for proposal parsing/validation/application
- Standard library: `argparse`, `json`, `sys`, `pathlib.Path`, `asyncio`, `os`, `shutil`

**Imported by:**
- The `main()` function is the console script entry point (likely wired in `pyproject.toml` as `reasons = "reasons_lib.cli:main"`). Nothing inside the package imports this module.

### 5. Flow

1. `main()` parses `sys.argv` into an `args` namespace
2. The `commands` dict resolves the subcommand string to a `cmd_*` function
3. The handler calls `api.*`, passing through CLI arguments
4. The API function returns a result dict
5. The handler formats and prints the result

For **derive**, the flow is more involved:
1. `cmd_derive` checks for `--exhaust` mode (loop up to `max_rounds`)
2. Each round calls `_derive_one_round`, which exports the network, builds a prompt, invokes a model CLI as a subprocess, parses proposals from the response, validates them against the current network, and either auto-applies or writes to file
3. The loop terminates when a round returns 0 (saturated) or -1 (error)

### 6. Invariants

- **Every command exits 1 on error** — no command silently swallows failures
- **`--db` defaults to `api.DEFAULT_DB`** — all commands share the same default database path
- **`what-if` is read-only** — the output explicitly states "database NOT modified"
- **`check-stale` exits 1 when stale nodes exist** — this makes it usable as a CI gate
- **`--any` flag semantics**: when set, SL justifications are expanded into one-per-premise (OR semantics); the CLI warns when 3+ premises are used without `--any`
- **`--exhaust` implies `--auto`** — auto-apply is checked via `args.auto or args.exhaust` in `_derive_one_round`
- **Model subprocess strips `CLAUDECODE` env var** — prevents recursive Claude Code invocation when deriving

### 7. Error Handling

Every command follows the same pattern:
```python
try:
    result = api.some_function(...)
except (KeyError, ValueError, ...) as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

The specific exceptions caught vary by command — `KeyError` for missing nodes, `ValueError` for invalid arguments, `FileExistsError` for `init` without `--force`, `FileNotFoundError` for missing import files. The derive path also catches `TimeoutError` and generic `Exception` from subprocess invocation, returning -1 to signal the caller.

No command returns error information to its caller; all errors terminate via `sys.exit(1)`.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The API layer that every CLI command delegates to; understanding its return-dict contracts is essential for maintaining the CLI
- [function] `reasons_lib/derive.py:build_prompt` — How the derivation prompt is constructed from the belief network, including filtering, sampling, and budget constraints
- [function] `reasons_lib/cli.py:_derive_one_round` — The most complex function in this file; worth understanding the subprocess invocation, timeout handling, and exhaust-loop interaction
- [file] `reasons_lib/network.py` — The core belief network data structure that `Storage` loads and `api` manipulates; understanding `recompute_all()` clarifies what `cmd_propagate` does
- [general] `what-if-simulation` — How `what_if_retract`/`what_if_assert` achieve read-only simulation while reporting cascading effects with depth information

## Beliefs

- `cli-is-pure-formatter` — Every `cmd_*` function delegates to `api.*` and only formats the returned dict; no business logic lives in the CLI layer (sole exception: `cmd_propagate` touches `Storage` directly)
- `check-stale-exits-nonzero` — `cmd_check_stale` calls `sys.exit(1)` when any stale nodes are found, making it usable as a CI/pre-commit gate
- `derive-strips-claudecode-env` — `_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess, preventing recursive Claude Code invocation
- `exhaust-implies-auto` — In `_derive_one_round`, proposals are auto-applied when either `args.auto` or `args.exhaust` is true; the `--exhaust` flag does not require the user to also pass `--auto`
- `all-errors-exit-one` — No CLI command propagates exceptions to the caller; all errors are printed to stderr and terminate with `sys.exit(1)`

