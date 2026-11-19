---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/cli.py

# `reasons/cli.py`

## Purpose

This file is the **command-line front end** for the Reason Maintenance System (RMS). Every CLI subcommand (`reasons add`, `reasons retract`, `reasons derive`, …) is registered in `main()` and dispatched to a thin `cmd_*` handler. Each handler's only job is:

1. Normalize a handful of `argparse` attributes (access tags, namespace, backend selection),
2. Call **one** function on `reasons.api`,
3. Pretty-print the returned `dict` to `stdout`, or convert exceptions into `stderr` messages + `sys.exit(1)`.

It contains **no business logic** — propagation, Merkle hashing, LLM invocation, DB selection, etc. all live in `reasons.api`, `reasons.storage`, `reasons.derive`, `reasons.llm`, etc. This file is intentionally a wide but shallow surface (~90 handlers). It also carries the shared plumbing (`_backend_kwargs`, `_require_sqlite`, `_parse_visible_to`, `_parse_tag_filters`) that every handler relies on.

---

## Key Components

### CLI plumbing helpers

| Symbol | Contract |
|---|---|
| `main()` | Builds the `argparse.ArgumentParser`, registers every subparser, parses `sys.argv`, and dispatches through the `commands` dict. Prints help + `sys.exit(1)` if no subcommand supplied. |
| `_backend_kwargs(args)` | Returns `{"pg_conninfo":…, "domain_id":…}` if `--pg`/`REASONS_PG_CONNINFO` is set (requires `--domain-id`/`--project-id`/env), otherwise `{"db_path": args.db}`. Called by virtually every `cmd_*` as `**_backend_kwargs(args)`. |
| `_require_sqlite(args, command_name)` | Guard used by ~25 commands that are SQLite-only. If PG is configured, prints an error and `sys.exit(1)`. |
| `_parse_visible_to(args)` | Splits `--visible-to "tag1, tag2"` into `["tag1","tag2"]` or returns `None` for no restriction. |
| `_parse_tag_filters(args)` | Groups repeated CLI options (`--topic`, `--origin`, `--priority`, `--stream`, `--tag`, `--workflow-status`) into the `{key: [vals]}` shape `api.search`/`api.list_nodes` expect. |
| `_parse_tags(args)` | Splits `--tags "a,b,c"` for the `propose-*` commands. |

### Shared output formatters

- `_print_cascade(result)` — separates `went_out` (`[-]`) from `went_in` (`[+]`). Used by `cmd_retract`, `cmd_assert`.
- `_print_restoration_hints(hints)` — prints a suggestion to re-add a multi-premise SL with `--any` after an SL was invalidated.
- `_print_what_if_results(result, action, node_id)` — groups hypothetical cascades by `depth` and appends `(N dependents)`.
- `_warn_multi_premise(premise_count, any_mode)` — non-fatal tip printed after `add`/`add-justification` when ≥3 SL premises were used without `--any`.

### Derive helpers (the only non-trivial control flow)

- `_derive_one_round(args, round_num=None, report_state=None, cluster_cache=None, prompt_template=None)` — runs a single `derive` pass: exports network → `build_prompt` → `invoke_model` → `parse_proposals` → `validate_proposals` → either `apply_proposals` (auto) or `write_proposals_file` (review). Returns `+N` count, `0` (saturated/proposals written), or `-1` (fatal error). Mutates `report_state["rounds"]` and calls `_write_derive_report(..., "partial")`.
- `_write_derive_report(report_state, status)` — serializes the accumulated `report_state` to `reviews/derive-*.json`.

### `cmd_*` handlers (~90)

Grouped by domain:

- **Core truth maintenance**: `cmd_init`, `cmd_add`, `cmd_add_justification`, `cmd_remove_justification`, `cmd_retract`, `cmd_assert`, `cmd_propagate`, `cmd_convert_to_premise`, `cmd_summarize`, `cmd_supersede`.
- **Reads**: `cmd_status`, `cmd_show`, `cmd_explain`, `cmd_trace`, `cmd_trace_access_tags`, `cmd_list`, `cmd_list_gated`, `cmd_compact`, `cmd_search`, `cmd_lookup`, `cmd_search_sources`, `cmd_topics`, `cmd_log`.
- **Tags/metadata/repos**: `cmd_tag`, `cmd_untag`, `cmd_tags`, `cmd_set_metadata`, `cmd_get_metadata`, `cmd_add_source`, `cmd_add_repo`, `cmd_repos`, `cmd_namespaces`.
- **Defeaters / dialectics**: `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_migrate_defeaters`, `cmd_classify_defeaters`, `cmd_challenge`, `cmd_defend`, `cmd_nogood`.
- **Simulation**: `cmd_what_if` (retract / assert / supersede).
- **Hive proposals**: `cmd_propose_retract`, `cmd_propose_supersede`, `cmd_propose_add`, `cmd_propose_nogood`, `cmd_proposals`, `cmd_proposal`, `cmd_accept_proposal`, `cmd_reject_proposal`, `cmd_withdraw_proposal`.
- **Import/export (files, HF, API)**: `cmd_import_agent`, `cmd_sync_agent`, `cmd_import_beliefs`, `cmd_import_json`, `cmd_import_hf`/`cmd_pull`, `cmd_publish`, `cmd_import_api`, `cmd_export_api`, `cmd_export`, `cmd_export_markdown`, `cmd_export_card`.
- **Provenance**: `cmd_hash_sources`, `cmd_check_stale`, `cmd_check_integrity`, `cmd_backfill_hashes`, `cmd_pin_sources`, `cmd_pin_update`, `cmd_pin_lines`.
- **LLM-powered review / repair**: `cmd_derive`, `cmd_accept`, `cmd_ask`, `cmd_cluster_list`, `cmd_deduplicate`, `cmd_contradictions`, `cmd_report`, `cmd_report_gated`, `cmd_verify`, `cmd_build_wiki`, `cmd_review_beliefs`, `cmd_review_justifications`, `cmd_review_premises`, `cmd_repair_premises`, `cmd_propose_update`, `cmd_repair_smuggled`, `cmd_repair`/`cmd_research` (alias), `cmd_list_negative`.

---

## Patterns

1. **Command pattern + dispatch table.** `main()` defines `parser = argparse.ArgumentParser(...)`; each subparser binds to a name; the final block builds `commands = {"add": cmd_add, ...}` and calls `commands[args.command](args)`. There's no `set_defaults(func=…)` — the dict is authoritative.

2. **Facade over `reasons.api`.** Each `cmd_*` is exactly one `api.*` call. Result **dicts** are the shared contract (`{"node_id": …, "truth_value": …, "changed": [...]}` etc.) — not exceptions or dataclasses.

3. **Deferred imports for expensive modules.** Heavy / optional dependencies are imported **inside** the handler so commands without LLM / embeddings / MCP don't pay startup cost:
   ```python
   from .ask import ask
   from .mcp_client import McpBridge
   from .derive import build_prompt, parse_proposals, …
   from .llm import invoke_model, reset_cost_tracker, format_cost_summary
   from .cluster import ClusterCache
   ```

4. **Backend polymorphism via kwargs.** `_backend_kwargs` returns either `{"db_path": …}` or `{"pg_conninfo": …, "domain_id": …}`. API functions accept `**kwargs` and dispatch (see `reasons/api.py`). No subclass of `Api` per backend at the CLI layer.

5. **Error-to-exit translation.** Handlers use a small set of `except` clauses (`KeyError`, `ValueError`, `IndexError`, `PermissionError`, `FileNotFoundError`, `RuntimeError`, `FileExistsError`) and turn them into `print(f"Error: {e}", file=sys.stderr); sys.exit(1)`. Returning normally means success (`exit 0`).

6. **Stream discipline.** User payload → `stdout`; diagnostics / cost summaries / warnings / per-round progress → `stderr`. This lets `reasons export -o -`, `reasons search`, `reasons ask`, etc. be piped.

7. **Optional progress reports.** LLM review commands (`review-beliefs`, `review-premises`, `propose-update`, `derive`) write a JSON report into `--report-dir` (default `reviews/`) using an `on_batch`/`on_result` callback so an interrupted run still leaves partial data. `--no-report` disables this, `--dry-run` short-circuits side effects.

8. **Cost accounting guard.** Every LLM-using command calls `reset_cost_tracker()` at start and `format_cost_summary()` at end, printing the summary to `stderr`.

---

## Dependencies

**Imported by `cli.py`:**
- Stdlib: `argparse`, `json`, `os`, `sys`, `pathlib.Path`, `importlib.metadata.version`.
- Internal, always: `from . import api`.
- Internal, deferred: `.ask`, `.mcp_client`, `.derive`, `.llm`, `.cluster`, `.propose_update`.

**Imported by (per repo context):**
- `reasons/mcp_client.py`, `reasons_lib/cli.py`, `reasons_lib/mcp_client.py`.
- Test hit list: `test_cli`, `test_ask`, `test_ask_mcp`, `test_build_wiki`, `test_contradictions`, `test_derive`, `test_example`, `test_hf_default_org`, `test_pg_dispatch`, `test_repair`, `test_repair_premises`, `test_report_belief`, `test_report_gated`, `test_research`, `test_review`, `test_review_premises`, `test_roundtrip`, `test_search_sources`, `test_source_type`, `test_topics`, `test_verify`. Almost every LLM-adjacent feature is exercised **through** this file (the tests invoke `main()` with `sys.argv` patched).

---

## Flow

```
main()
 ├─ build ArgumentParser
 ├─ add global flags: --version, --db, --pg, --domain-id, --project-id
 ├─ sub.add_parser(name) for every command
 ├─ args = parser.parse_args()
 ├─ if args.command is None: print_help(); sys.exit(1)
 └─ commands[args.command](args)
     └─ cmd_*()
         ├─ normalize: access_tags, visible_to, tag filters, backend kwargs
         ├─ result = api.<fn>(…, **_backend_kwargs(args))
         ├─ print lines to stdout
         └─ except SpecificError → stderr + sys.exit(1)
```

A canonical handler (`cmd_retract`) shows the shape:

```python
try:
    result = api.retract_node(args.node_id, reason=args.reason or "",
                              **_backend_kwargs(args))
except KeyError as e:
    print(f"Error: {e}", file=sys.stderr); sys.exit(1)
if not result["changed"]:
    print(f"{args.node_id} is already OUT")
else:
    print(f"Retracted {args.node_id}")
    _print_cascade(result)
    if result.get("restoration_hints"):
        _print_restoration_hints(result["restoration_hints"])
```

`cmd_derive` is the only handler with real orchestration (exhaust loop; single-round vs. `--dry-run`; mutating `report_state`; cost tracking).

---

## Invariants

1. **PG requires domain.** When `_backend_kwargs` sees PG configured (flag or `REASONS_PG_CONNINFO`), a non-empty domain id (`--domain-id`, `--project-id`, `REASONS_DOMAIN_ID`, `REASONS_PROJECT_ID`) must be present — otherwise it exits 1 with a `stderr` message **before** any API call.

2. **SQLite-only commands.** The following are guarded by `_require_sqlite`, or an inline equivalent check, and refuse to run against PostgreSQL: `check-integrity`, `backfill-hashes`, `pin-sources`, `pin-update`, `pin-lines`, `ask`, `cluster-list`, `deduplicate`, `derive`, `accept`, `review-beliefs`, `review-justifications`, `review-premises`, `repair-premises`, `propose-update`, `repair-smuggled`, `repair`, `contradictions`, `build-wiki`, plus `mark-duplicate`, `mark-superseded`, `defeat-justification`, `defeat-with-scope`, `migrate-defeaters`, `classify-defeaters` (inline `os.environ.get("REASONS_PG_CONNINFO")` check).

3. **Mutual exclusions inside `derive`.** `--cluster/--intra-cluster` cannot combine with `--sample`, and `--cluster` cannot combine with `--intra-cluster`. Both checks `sys.exit(1)` before touching the DB.

4. **`supersede` argument XOR.** `cmd_supersede` requires exactly one of `new_id` or `--text` — passing both, or neither, exits 1.

5. **`search-sources` uses a *separate* DB.** Its `--db` is the FTS5 *chunks* DB, not the reasons DB. Lookup goes through `reasons.ask.search_source_chunks`, not `reasons.api`.

6. **`status` summary is always printed** (IN/OUT/total, premises/derived, namespaces); the full node listing appears only with `--all`, `--namespace`, `--status`, `--premises`, or `--limit`.

7. **`check-stale` exits 1 when any node is STALE** — flag this because every other command returns 0 unless something raised an exception.

8. **Stdout is data, stderr is diagnostics.** Do not print human chatter to stdout in `export*`, `search`, `lookup`, `ask`, `compact` — those are meant to be shell-pipable.

9. **Handlers are pure w.r.t. `args`.** No handler mutates `args` or crosses back into another `cmd_*` (except aliases: `cmd_pull` → `cmd_import_hf`, `research` → `cmd_repair`).

---

## Error Handling

- **Bare `except (KeyError, ValueError, IndexError)` → `sys.exit(1)`.** This is the "not found / bad input" tier. The `Exception` message is *the entire error surface* — no tracebacks.
- **`FileExistsError`** appears only in `cmd_init` and `cmd_import_hf`/`cmd_import_api`, plus a helpful "Use `--force` to reinitialize" hint in `cmd_init`.
- **`PermissionError`** is caught in `cmd_show`, `cmd_explain`, `cmd_trace`, `cmd_trace_access_tags` — printed as `Access denied: …`.
- **`FileNotFoundError`** wraps import handlers (`import-agent`, `sync-agent`, `import-beliefs`, `import-json`) and derive's inner loop.
- **`RuntimeError`** wraps the HF/API import/publish path.
- **`subprocess.TimeoutExpired` + generic `Exception`** are caught inside `_derive_one_round` and return `-1`, which causes `cmd_derive` to `sys.exit(1)` — so LLM timeouts don't leak tracebacks.
- **Silent failures are deliberately avoided** except for per-item errors in repair/review loops: individual `api.retract_node`/`api.defeat_justification` calls inside `cmd_review_beliefs` are wrapped in `try/except Exception` and printed as `ERROR retracting {id}: …` to `stderr`, but processing continues.
- **`cmd_check_stale` returns exit 1 when stale nodes exist**, intentionally diverging from the "exception → exit 1" model so CI can gate on freshness.

## Topics to Explore

- [file] `reasons/api.py` — The façade every `cmd_*` calls into; the real contract (return-dict shapes, backend dispatch) lives here.
- [file] `reasons/derive.py` — `build_prompt`, `parse_proposals`, `validate_proposals`, `apply_proposals` — the pipeline `_derive_one_round` orchestrates.
- [file] `reasons/storage.py` — How `db_path` vs `pg_conninfo`/`domain_id` is actually consumed by the API layer.
- [function] `reasons/llm.py:format_cost_summary` — How token cost tracking works across the LLM-running subcommands.
- [file] `tests/test_cli.py` — The canonical reference for expected `argparse` behavior, exit codes, and stdout formatting.

## Beliefs

- `cli-handlers-are-thin-api-wrappers` — Every `cmd_*` function other than `_derive_one_round` (and its helpers) performs exactly one `reasons.api` call followed by dict-formatted `stdout` output.
- `pg-backend-requires-domain-id` — `_backend_kwargs` refuses to return PostgreSQL kwargs unless a domain id is supplied via flag or the `REASONS_DOMAIN_ID`/`REASONS_PROJECT_ID` env var, exiting with code 1 otherwise.
- `sqlite-only-commands-guarded-by-require-sqlite` — Commands that touch the local filesystem, Merkle hashes, embeddings, or LLM-specific workflows call `_require_sqlite` (or the equivalent inline `REASONS_PG_CONNINFO` check) and abort if PostgreSQL is configured.
- `main-dispatches-by-command-string-dict` — `main()` routes subcommand execution through a `commands: dict[str, Callable]` table rather than `argparse.set_defaults(func=...)`.
- `derive-cluster-and-sample-are-mutually-exclusive` — `cmd_derive` exits with code 1 if both `--cluster`/`--intra-cluster` and `--sample` are supplied, before making any API call.
- `check-stale-exits-nonzero-on-stale-nodes` — Unlike the general error-to-exit-1 convention, `cmd_check_stale` deliberately exits 1 when any source is stale or deleted, enabling CI freshness gates.
