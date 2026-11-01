---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/cli.py

# `reasons/cli.py` — CLI for the Reason Maintenance System

## 1. Purpose

This file is the **sole CLI entry point** for the entire `ftl-reasons` project. It owns the responsibility of:

- Defining the full command surface (~100 subcommands) via `argparse`
- Translating parsed CLI arguments into `reasons.api` function calls
- Formatting the resulting dictionaries into human-readable terminal output
- Handling the backend dispatch decision (SQLite vs. PostgreSQL)
- Enforcing preconditions that are CLI-specific (e.g., "this command doesn't work with `--pg`")

The docstring states it clearly: *"Thin wrappers around reasons.api — each command calls an api function and formats the result dict for terminal output."* The actual belief-network logic (truth propagation, dependency-directed backtracking, justification semantics) lives in `api.py` and its dependencies. This file contains **no business logic** beyond argument parsing and output formatting.

## 2. Key Components

### Entry Point: `main()` (bottom of file)

- Builds the top-level `argparse.ArgumentParser` with global flags: `--db`, `--pg`, `--domain-id`, `--project-id`
- Registers ~100 subparsers via `sub.add_parser(...)`
- Dispatches to a handler via the `commands` dict (a flat string→function mapping)
- If no command is given, prints help and exits with code 1

### Handler Functions: `cmd_<name>(args)`

Every handler follows the same shape:

```python
def cmd_retract(args):
    try:
        result = api.retract_node(args.node_id, reason=args.reason or "", **_backend_kwargs(args))
    except KeyError as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)
    # ... format and print result ...
```

Notable handlers by complexity:

| Function | What it does |
|----------|-------------|
| `cmd_init` | Initializes a DB (SQLite or PG domain); handles `FileExistsError` specially |
| `cmd_add` | Creates a node with optional SL/CP justifications, `--any` mode, access tags |
| `cmd_derive` | Orchestrates LLM-based belief derivation with optional `--exhaust` loop, clustering, cost tracking |
| `cmd_review_beliefs` | LLM-powered review of derived beliefs; can auto-retract or auto-defeat invalid ones |
| `cmd_review_premises` | LLM-powered factual accuracy check of premises against sources |
| `cmd_repair` | Multi-strategy repair: search-and-link, soften, abandon |
| `cmd_contradictions` | Detects contradictions via LLM; supports plan/apply two-phase workflow |
| `cmd_what_if` | Read-only simulation of retract/assert/supersede cascades |
| `cmd_ask` | Full RAG pipeline: FTS search + optional MCP bridges + LLM synthesis |
| `cmd_verify` | Verifies beliefs against source docs; can retract STALE ones |

### Utility Helpers

- **`_backend_kwargs(args)`** — The critical dispatch function. Returns `{"db_path": args.db}` for SQLite mode, or `{"pg_conninfo": ..., "domain_id": ...}` for PG mode. Reads env vars `REASONS_PG_CONNINFO` and `REASONS_DOMAIN_ID` as fallbacks. Exits with error if PG is requested without a domain ID.

- **`_require_sqlite(args, command_name)`** — Guard that exits with an error message if `--pg` is active. Used by ~30 commands that have no PG implementation.

- **`_parse_visible_to(args)`** — Splits a comma-separated `--visible-to` tag list into a Python list, or returns `None`.

- **`_warn_multi_premise(premise_count, any_mode)`** — Prints a tip when a node has ≥3 premises and `--any` wasn't used, suggesting the user might want OR semantics.

- **`_print_cascade(result)`** — Shared formatter for the `went_out`/`went_in` lists in propagation results.

- **`_print_what_if_results(result, action, node_id)`** — Shared formatter for simulation output, grouping by depth.

- **`_parse_tag_filters(args)`** — Builds a `dict[str, list[str]]` from repeated `--topic`, `--origin`, `--priority`, `--stream`, `--tag` flags for search/list filtering.

### LLM Cost Tracking Pattern

Commands that invoke an LLM follow a consistent three-step pattern:

```python
from .llm import reset_cost_tracker, format_cost_summary
reset_cost_tracker()
# ... call api (which internally calls invoke_model) ...
cost = format_cost_summary()
if cost:
    print(f"  {cost}", file=sys.stderr)
```

## 3. Patterns

**Command Pattern (flat)**: Each subcommand is a standalone `cmd_*` function registered in a dict. There's no base class, no visitor, no plugin system — just a flat mapping. This is intentional: the CLI is the stable interface, and adding a command is as simple as adding a function + a subparser block + one dict entry.

**Facade / Thin Wrapper**: The file never manipulates beliefs directly. Every mutation goes through `api.*`. The only exception is `cmd_derive`/`cmd_accept` which import from `reasons.derive` and `reasons.llm` for the prompt-building and model-invocation pipeline — but even then, the actual DB writes go through `api.export_network` and `apply_proposals`.

**Two-Phase Workflows**: Several commands implement a "propose → review → apply" pattern:
- `deduplicate` writes a plan file → user edits → `deduplicate --accept FILE`
- `contradictions` writes a plan file → user edits → `contradictions --accept FILE`
- `derive` without `--auto` writes proposals → user reviews → `accept FILE`
- `propose-update` with `--store` writes to the proposals table → `accept-proposal`

This is a deliberate safety pattern for destructive operations.

**Access Control Filtering**: The `--visible-to TAG,TAG` flag appears on ~20 commands. It's parsed by `_parse_visible_to()` and passed as `visible_to` to the API. The API enforces that a node's `access_tags` must be a subset of the provided tags. This is a multi-tenant / multi-agent data isolation mechanism.

**Backend Abstraction via kwargs**: `_backend_kwargs()` returns a dict that is unpacked into every API call. This means the API functions accept either `db_path=...` or `pg_conninfo=..., domain_id=...`. The CLI never branches on backend type (except for the `_require_sqlite` guards).

**Report Generation**: LLM-heavy commands (`derive`, `review-beliefs`, `review-premises`, `repair-premises`, `propose-update`) write timestamped JSON reports to a `reviews/` directory. They support `--no-report` to suppress, and use an `on_batch`/`on_result` callback to write partial reports incrementally (useful for long-running operations).

## 4. Dependencies

### What this file imports:

| Dependency | Usage |
|-----------|-------|
| `reasons.api` | **Primary** — all belief operations, ~90% of handler logic |
| `reasons.llm` | `invoke_model`, `reset_cost_tracker`, `format_cost_summary` (for `derive`, `ask`, `verify`, etc.) |
| `reasons.derive` | `build_prompt`, `parse_proposals`, `validate_proposals`, `apply_proposals`, `write_proposals_file` |
| `reasons.ask` | `ask()`, `search_source_chunks()` |
| `reasons.mcp_client` | `McpBridge` for MCP server connections in `cmd_ask` |
| `reasons.cluster` | `ClusterCache` for semantic clustering in `derive` and `cluster-list` |
| `reasons.propose_update` | `format_proposals_file` for markdown output |
| `argparse`, `json`, `os`, `sys`, `pathlib` | Standard library |
| `importlib.metadata` | `version('ftl-reasons')` for `--version` |

### What imports this file:

- **`reasons_lib/cli.py`** — A duplicate/mirror copy (the `reasons_lib/` directory appears to be a vendored or legacy copy)
- **`reasons/mcp_client.py`** — Likely imports `cli` for the MCP server to expose CLI commands as MCP tools
- **~20 test files** in `tests/` — `test_cli.py`, `test_ask.py`, `test_derive.py`, `test_review.py`, `test_verify.py`, etc.

## 5. Flow

### Simple command (e.g., `reasons retract my-belief --reason "wrong"`):

```
main()
  → argparse parses "retract" as command, "my-belief" as node_id, "--reason" as reason
  → looks up commands["retract"] → cmd_retract
  → _backend_kwargs(args) → {"db_path": "reasons.db"}
  → api.retract_node("my-belief", reason="wrong", db_path="reasons.db")
    [api internally: marks node OUT, runs DDBT cascade, recomputes dependents]
  → prints "Retracted my-belief" + cascade details
```

### Complex LLM command (e.g., `reasons review-beliefs --auto-retract -m claude`):

```
main()
  → cmd_review_beliefs(args)
  → reset_cost_tracker()
  → builds report_path, defines _build_report/_write_report closures
  → api.review_beliefs(belief_ids=None, model="claude", on_batch=callback, ...)
    [api internally: loads derived beliefs, batches them, calls LLM per batch,
     collects verdicts, calls on_batch after each batch for partial report]
  → prints INVALID/INSUFFICIENT/UNNECESSARY findings
  → writes final JSON report
  → if --auto-retract: iterates invalid beliefs, calls api.retract_node for each
  → format_cost_summary() → prints to stderr
```

### The `--exhaust` derive loop:

```
cmd_derive
  → validates flag combinations (cluster vs sample exclusivity)
  → optionally loads ClusterCache
  → for round_num in 1..max_rounds:
      _derive_one_round(...)
        → api.export_network() [load current state]
        → build_prompt(...) [construct LLM prompt from beliefs]
        → invoke_model(prompt) [call LLM]
        → parse_proposals(response) [extract structured proposals]
        → validate_proposals(proposals, nodes) [check against existing network]
        → apply_proposals(valid) [write to DB]
        → returns count added (0 = saturated)
      if added == 0: break
  → writes final report
  → prints cost
```

## 6. Invariants

1. **PG requires domain**: `_backend_kwargs()` will `sys.exit(1)` if `--pg` is set but no `--domain-id` is provided (via flag or env). This prevents accidentally targeting the wrong database.

2. **SQLite-only commands are guarded**: ~30 commands call `_require_sqlite(args, name)` at the top. If `REASONS_PG_CONNINFO` is set or `--pg` is passed, they refuse to run. This includes all LLM-heavy commands, integrity checks, and pinning operations.

3. **`--any` mode changes justification semantics**: When `--any` is passed to `add` or `add-justification`, an SL with N antecedents becomes N separate justifications (OR) instead of one justification with N antecedents (AND). The `_warn_multi_premise` helper nudges users toward this when they have ≥3 premises without `--any`.

4. **Cascade propagation is always reported**: After any truth-value mutation (retract, assert, defeat, supersede), the result includes `changed`, `went_out`, `went_in` lists. The CLI always prints these — it never silently swallows cascade effects.

5. **Read-only simulation for `what-if`**: The `cmd_what_if` handler explicitly prints "(database NOT modified)". The API is expected to perform the cascade computation without persisting.

6. **Proposal lifecycle is atomic**: A proposal goes `pending → accepted|rejected|withdrawn|stale`. `accept-proposal` either applies the change or marks it stale (if the target no longer exists). It never partially applies.

7. **Access tags are subset-checked, not equality-checked**: A node with `access_tags=["internal"]` is visible to a user with `--visible-to internal,public` (subset), but NOT to `--visible-to secret` (not a subset).

## 7. Error Handling

The file uses a **uniform try/except/exit pattern** with no custom exception types:

```python
try:
    result = api.some_function(...)
except (KeyError, ValueError) as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

**Exception → meaning mapping:**

| Exception | Meaning |
|-----------|---------|
| `KeyError` | Node ID not found in the network |
| `ValueError` | Invalid argument combination or business rule violation |
| `IndexError` | Justification index out of range |
| `PermissionError` | Access tag check failed (node not visible to requested tags) |
| `FileExistsError` | DB already exists (from `init`) or import conflict |
| `FileNotFoundError` | Input file (beliefs.md, proposals, scope file) doesn't exist |
| `RuntimeError` | LLM call failed, HF API error, network error |
| `subprocess.TimeoutExpired` | LLM model call timed out |

**Error output conventions:**
- All errors go to `sys.stderr`
- Error messages are prefixed with `"Error: "` or the exception context
- Exit code is always `1` for errors, `0` for success
- `cmd_check_stale` exits `1` if stale nodes are found (useful in CI pipelines)
- `cmd_check_integrity` exits `1` if mutations are detected

**Notable: no error is "swallowed."** Every `except` block either prints and exits, or (in `cmd_derive`'s model invocation) prints to stderr and returns `-1` to signal failure to the caller, which then calls `sys.exit(1)`.

**One exception to the pattern**: `cmd_what_if` and `cmd_ask` use `try/finally` to ensure MCP bridge connections are closed even on error:

```python
try:
    ...
finally:
    for bridge in mcp_servers:
        bridge.close()
```

---

## Topics to Explore

- [file] `reasons/api.py` — The actual implementation behind every `cmd_*` function; understanding its function signatures and return dict shapes is prerequisite to understanding the CLI
- [function] `reasons/cli.py:_derive_one_round` — The most complex handler; it orchestrates prompt building, LLM invocation, proposal parsing, validation, and application in a single round of the derive loop
- [general] `pg-vs-sqlite-dispatch` — How `_backend_kwargs()` and `_require_sqlite()` gate functionality; which operations are PG-compatible and which aren't, and why
- [file] `reasons/derive.py` — The `build_prompt`, `parse_proposals`, `validate_proposals`, and `apply_proposals` functions that `cmd_derive` delegates to for the actual derivation logic
- [file] `tests/test_cli.py` — The test suite for this file; reveals expected output formats, error paths, and edge cases that aren't obvious from reading the handlers alone

## Beliefs

- `cli-is-a-thin-facade-over-api` — Every `cmd_*` handler calls exactly one (or a small fixed set) of `reasons.api` functions and contains no belief-network logic itself; all truth propagation, DDBT, and justification semantics live in `api.py` and its submodules
- `pg-mode-requires-domain-id` — `_backend_kwargs()` will call `sys.exit(1)` if a PostgreSQL connection string is provided (via `--pg` or `REASONS_PG_CONNINFO`) but no domain ID is available (via `--domain-id`, `--project-id`, or `REASONS_DOMAIN_ID`), preventing writes to an unscoped database
- `visible-to-enables-access-isolation` — The `--visible-to` flag is parsed into a list and passed as `visible_to` to API calls; the API enforces that a node's `access_tags` must be a **subset** of the provided tags, enabling multi-agent data isolation without separate databases
- `destructive-commands-use-two-phase-workflow` — `deduplicate`, `contradictions`, `derive` (without `--auto`), and `propose-update` (with `--store`) all write a plan/proposal artifact first and require an explicit second invocation (`--accept`, `accept`, `accept-proposal`) to apply changes, preventing irreversible LLM-driven mutations
- `llm-cost-tracking-is-mandatory-for-model-commands` — Every command that invokes an LLM calls `reset_cost_tracker()` before the API call and `format_cost_summary()` after; the cost summary is printed to `stderr` so it doesn't interfere with stdout output that may be piped to other tools
