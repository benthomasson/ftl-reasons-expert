# File: reasons_lib/cli.py

**Date:** 2026-05-08
**Time:** 14:02

## `reasons_lib/cli.py` — Explanation

### 1. Purpose

This is the CLI entry point for the `reasons` command — a Truth Maintenance System (TMS) tool. It owns **argument parsing** and **terminal output formatting**. It deliberately contains no business logic: every `cmd_*` function calls into `reasons_lib.api`, receives a result dict, and formats it for human consumption. The docstring says it plainly: *"Thin wrappers around reasons_lib.api."*

The `main()` function at the bottom is the console script entry point (registered via `pyproject.toml`), building a single `argparse` parser with ~40 subcommands.

### 2. Key Components

**`main()`** — Constructs the argparse parser, dispatches to command functions via a `commands` dict keyed by subcommand name. Every subcommand gets `--db` for database path selection (defaulting to `api.DEFAULT_DB`).

**`cmd_*` functions** — One per subcommand. Each follows the same contract:
- Takes an `args` namespace from argparse
- Calls an `api.*` function, passing parsed arguments
- Formats the returned dict for stdout
- Catches specific exceptions (`KeyError`, `ValueError`, `FileNotFoundError`, `PermissionError`) and exits with code 1 on failure

**Notable command groups:**

| Group | Commands | Purpose |
|-------|----------|---------|
| Core CRUD | `add`, `retract`, `assert`, `show`, `update` | Create/modify/inspect nodes |
| Reasoning | `explain`, `trace`, `what-if`, `propagate` | Inspect justification chains and simulate changes |
| Dialectical | `challenge`, `defend`, `nogood` | Argumentation and contradiction recording |
| Import/Export | `import-agent`, `sync-agent`, `import-beliefs`, `import-json`, `export`, `export-markdown` | Data interchange |
| LLM-powered | `derive`, `review-beliefs`, `contradictions`, `ask`, `compact`, `list-negative` | AI-assisted reasoning, review, and querying |
| Maintenance | `check-stale`, `hash-sources`, `deduplicate`, `supersede` | Network hygiene |

**Helper functions:**

- `_print_cascade(result)` — Formats retraction/assertion cascade output, splitting `went_out` from `went_in`.
- `_print_restoration_hints(hints)` — When a multi-premise SL node goes OUT, prints actionable hints about surviving premises and how to restore with `--any`.
- `_warn_multi_premise(premise_count, any_mode)` — Prints a tip when an SL has 3+ premises and `--any` wasn't used, nudging the user toward OR-mode if appropriate.
- `_print_what_if_results(result, action, node_id)` — Shared formatter for `what-if retract` and `what-if assert`, showing depth-layered cascades.
- `_parse_visible_to(args)` — Extracts comma-separated access tags from `--visible-to` into a list (or `None`).
- `_derive_one_round(args, ...)` — Runs a single derive iteration (build prompt, invoke LLM, parse/validate/apply proposals). Used by `cmd_derive` for both single-round and `--exhaust` mode.
- `_write_derive_report(report_state, status)` — Serializes derive run metadata to a JSON report file.

### 3. Patterns

**Thin CLI / Fat API**: Every command is a thin shell — parse args, call API, format output. This makes the API independently testable and usable programmatically.

**Result-dict protocol**: The API layer returns plain dicts (not domain objects). The CLI destructures them with `result['key']` access. This keeps the serialization boundary clean.

**Consistent error handling**: Almost every `cmd_*` wraps its API call in `try/except`, prints to stderr, and calls `sys.exit(1)`. The exception types map to failure modes: `KeyError` = node not found, `ValueError` = invalid input, `FileNotFoundError` = missing file, `PermissionError` = access-tag violation.

**Lazy imports**: Heavy dependencies (`derive`, `ask`, `cluster`, `llm`) are imported inside the functions that need them, not at module level. This keeps `reasons status` fast even when torch/sentence-transformers aren't installed.

**Dispatch table**: `main()` uses a `commands` dict mapping subcommand strings to functions, rather than `set_defaults(func=...)`. This keeps all routing visible in one place.

**Report accumulation**: `cmd_derive` and `cmd_review_beliefs` build up a `report_state` dict across rounds/batches, writing partial JSON reports after each step. This survives crashes mid-run.

### 4. Dependencies

**Imports:**
- `argparse`, `json`, `sys`, `pathlib.Path` — standard library
- `importlib.metadata.version` — for `--version` flag
- `reasons_lib.api` — the sole business-logic dependency for most commands
- `reasons_lib.ask`, `reasons_lib.derive`, `reasons_lib.llm`, `reasons_lib.cluster` — lazy-imported for LLM-powered commands

**Depended on by:**
- `tests/test_cli.py`, `tests/test_derive.py`, `tests/test_ask.py`, `tests/test_review.py`, `tests/test_contradictions.py` — test files that exercise CLI functions directly
- Registered as a console script entry point (`reasons = reasons_lib.cli:main` in pyproject.toml)

### 5. Flow

1. User runs `reasons <subcommand> [args]`.
2. `main()` parses args, looks up the command in the dispatch dict, calls `cmd_<subcommand>(args)`.
3. The `cmd_*` function extracts parameters from `args`, calls `api.*()`, and receives a result dict.
4. Output is printed to stdout (data) or stderr (errors, progress).
5. For LLM commands (`derive`, `review-beliefs`, `contradictions`), the flow is: load network → build prompt → invoke model via `llm.invoke_model()` → parse response → validate → optionally apply changes. `cmd_derive` with `--exhaust` loops this until saturation or max rounds.

**`cmd_derive` specifically:**
- Single round: calls `_derive_one_round()` once.
- Exhaust mode: loops `_derive_one_round()` up to `--max-rounds` times, stopping when 0 new proposals are returned.
- Each round: exports the network fresh, builds a prompt with optional filtering (topic, depth, clustering, sampling), invokes the model, parses proposals, validates against the current network, and either writes a proposals file or auto-applies.

### 6. Invariants

- **Every subcommand maps to exactly one `cmd_*` function** — the dispatch dict in `main()` is exhaustive.
- **`--db` is always available** — added to the top-level parser, inherited by all subcommands.
- **`--cluster` and `--sample` are mutually exclusive** in `cmd_derive` — enforced with an explicit check and `sys.exit(1)`.
- **Access-tag filtering** (`--visible-to`) is threaded through to the API for commands that read nodes: `status`, `show`, `explain`, `trace`, `export`, `list`, `compact`, `search`.
- **Dry-run semantics**: `what-if` never modifies the database; `derive --dry-run` prints the prompt without invoking the model; `review-beliefs --dry-run` reports findings without retracting.
- **`_derive_one_round` returns a sentinel**: positive = beliefs added, 0 = saturated, negative = error. This drives the exhaust loop's termination logic.

### 7. Error Handling

- **API exceptions → stderr + exit(1)**: `KeyError` (node not found), `ValueError` (bad input), `FileNotFoundError` (missing file), `PermissionError` (access denied). Each command catches the specific exceptions its API call might raise.
- **Model failures in derive**: `subprocess.TimeoutExpired`, `FileNotFoundError` (model binary missing), and generic `Exception` are caught in `_derive_one_round`, printed to stderr, and returned as -1 to signal the caller.
- **No-op cases printed, not errored**: "already OUT", "already IN", "no nodes", "no matching nodes" — these produce informative messages and return normally (exit 0).
- **Partial report writes**: `_write_derive_report` and `cmd_review_beliefs` write partial JSON reports after each batch/round, so crash recovery is possible.
- **`cmd_update` pre-validates**: checks that at least one of `--text`, `--source`, `--source-url` was provided before calling the API.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — The business-logic layer that every `cmd_*` function delegates to; understanding its return-dict contracts is essential for modifying CLI output
- [function] `reasons_lib/derive.py:build_prompt` — Controls what beliefs get included in derive prompts and how they're formatted; the filtering/sampling/clustering logic lives here
- [file] `reasons_lib/ask.py` — The `ask` command has its own module rather than going through `api`; worth understanding why it's structured differently
- [function] `reasons_lib/llm.py:invoke_model` — The model invocation abstraction that supports `claude`, `ollama:<model>`, and potentially others; shared by derive, review-beliefs, contradictions, and ask
- [general] `outlist-gate-propagation` — The CLAUDE.md documents a known issue where GATE beliefs aren't auto-re-evaluated when outlist nodes change; `cmd_assert` and `list-gated` are the manual workarounds

---

## Beliefs

- `cli-delegates-all-logic-to-api` — Every `cmd_*` function calls `api.*` for business logic and only handles argument parsing and output formatting; no TMS operations happen in `cli.py` itself
- `cli-lazy-imports-heavy-deps` — LLM-related modules (`derive`, `ask`, `cluster`, `llm`) are imported inside command functions, not at module top level, so lightweight commands like `status` don't pay the import cost
- `derive-one-round-return-protocol` — `_derive_one_round` returns positive for beliefs added, 0 for network-saturated, and negative for errors; `cmd_derive`'s exhaust loop depends on this contract
- `cli-exit-1-on-api-errors` — All command functions catch domain exceptions and call `sys.exit(1)`; no API error propagates as an unhandled traceback to the user
- `visible-to-threading` — Access-tag filtering via `--visible-to` is parsed by `_parse_visible_to` and passed through to the API layer; it is available on read commands (`show`, `explain`, `list`, `export`, `search`, `compact`) but not on write commands (`add`, `retract`)

