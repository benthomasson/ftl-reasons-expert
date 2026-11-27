---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search the full `reasons/cli.py` for `add_argument` and subparser setup to verify runtime flags.

## Explanation

### What was attempted

The task was to **scan the entire `reasons/cli.py` for `add_argument` calls and subparser setup** (`add_parser`, `add_subparsers`, `ArgumentParser`, `set_defaults`) in order to verify all CLI runtime flags. Two searches were run and two 500-line windows of the file were read.

### What the observations show

From `read_cli_imports` we can confirm that `reasons/cli.py` **does** import `argparse` (line ~4 in the first-half read) and calls `from . import api`. The module docstring in `read_cli_first_half` also explains the design: *"Thin wrappers around reasons.api — each command calls an api function and formats the result dict for terminal output."*

The two reads cover **lines 1–500** and **501–1000** of a **4015-line file**, i.e. ~25% of the total. Within those windows, every function is a `cmd_*` handler that consumes an already-parsed `args` object:

- `cmd_init(args)` uses `args.force`, `args.project_name` (via `getattr`) — lines ~16–29 of the first half.
- `cmd_add(args)` uses `args.node_id`, `args.text`, `args.sl`, `args.cp`, `args.unless`, `args.label`, `args.source`, `args.source_url`, `args.namespace`, `args.any`, `args.access_tags`, `args.example`, `args.source_type`, `args.accepted_pr` — lines ~41–68.
- `cmd_add_justification`, `cmd_remove_justification`, `cmd_retract`, `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, `cmd_classify_defeaters`, `cmd_assert`, `cmd_what_if`, `cmd_status`, `cmd_show`, `cmd_tag/untag/tags`, `cmd_add_source`, `cmd_explain`, `cmd_convert_to_premise`, `cmd_summarize`, `cmd_supersede`, `cmd_set/get_metadata`, `cmd_challenge`, `cmd_defend`, `cmd_nogood`, `cmd_trace_access_tags`, `cmd_trace`, `cmd_propagate`, `cmd_log`, `cmd_add_repo`, `cmd_repos`, `cmd_import_agent`, `cmd_sync_agent`, `cmd_import_beliefs`, `cmd_import_json`, `cmd_import_hf`, `cmd_pull` — all follow the same pattern of reading `getattr(args, ..., ...)`.
- `_backend_kwargs(args)` (referenced everywhere but defined **outside** the read windows) is clearly the function that materializes `--db` / `--pg` into kwargs.

### The key finding: the requested search did NOT succeed

Both searches returned:

- `find_add_argument_calls`: **0 matches**
- `find_subparser_and_parser_setup` (`add_parser|add_subparsers|ArgumentParser|set_defaults`): **0 matches**

This is **contradictory** with the rest of the evidence:

1. `argparse` is imported at the top of the file (see `read_cli_imports`).
2. Hundreds of `args.<flag>` reads exist in lines 1–1000 — those attributes must be defined by `add_argument` calls somewhere in the file.
3. `tests/test_cli.py` (lines 1–400) drives the parser through dozens of flags: `--db`, `--force`, `--sl`, `--access-tags`, `--any`, `--reason`, `--all`, `--visible-to`, `--premises`, `--status`, `--output`, plus subcommands `init`, `add`, `add-justification`, `retract`, `assert`, `status`, `show`, `explain`, `trace`, `trace-access-tags`, `list`, `search`, `lookup`, `export`. Those subcommands must be registered via `add_parser`/`add_subparsers`.

So the zero-match result almost certainly reflects a **search-scope or regex issue** (e.g. wrong glob, wrong workspace, a filtering layer that dropped the results, or patterns that didn't include the exact syntax such as `subparsers.add_parser(... [truncated]`) — not the absence of parser setup in the file.

### What we can infer about runtime flags from the handlers (partial list)

Even without the parser definition, the handler reads pin down a large part of the flag surface for the commands visible in lines 1–1000:

| Command | Flags referenced in handler |
|---|---|
| `init` | `--force`, `--project-name` (`getattr(args,"project_name","")`) |
| `add` | `--sl`, `--cp`, `--unless`, `--label`, `--source`, `--source-url`, `--namespace`, `--any`, `--access-tags`, `--example`, `--source-type`, `--accepted-pr` |
| `add-justification` | `--sl`, `--cp`, `--unless`, `--label`, `--namespace`, `--any` |
| `remove-justification` | `index` (positional?), `--db` |
| `retract` | `--reason` |
| `mark-duplicate` | `source_id`, `canonical_id`, `--db` (rejects `--pg`) |
| `mark-superseded` | `old_id`, `new_id`, `--db` |
| `defeat-justification` | `justification_index`, `--type`, `--defeater-id`, `--reason-type` |
| `defeat-with-scope` | `--scope-file`, `--type`, `--defeater-id`, `--reason-type` |
| `migrate-defeaters` | `--apply`, `node_ids` |
| `classify-defeaters` | `--apply`, `--type`, `--model`, `--timeout` |
| `what-if` | `action` (subaction), `--text`, `--new-id` |
| `status` | `--all`, `--namespace`, `--status`, `--premises`, `--limit`, `--offset`, `--format`, `--visible-to` |
| `show` / `explain` / `trace` / `trace-access-tags` | `--visible-to`, `--format` |
| `tag`/`untag` | `tags` (positional / nargs?) |
| `add-source` | `source_ref`, `--type`, `--url`, `--label` |
| `summarize` | `summary_id`, `text`, `--over`, `--source` |
| `supersede` | `new_id`, `--text`, `--id`, `--transitive` |
| `set-metadata` | `key`, `value` |
| `get-metadata` | `--key` |
| `challenge` / `defend` | `target_id`, `reason`, `--id`, `challenge_id` |
| `nogood` | `node_ids` |
| `log` | `--last` |
| `add-repo` | `name`, `path` |
| `import-agent` / `sync-agent` | `--agent-name`, `--beliefs-file`, `--nogoods-file`, `--only-in` |
| `import-json` | `json_file` |
| `import-hf` / `pull` | `--repo-id`, `--init`, `--token` |

`cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, `cmd_classify_defeaters` all explicitly check `getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO")` — so a global `--pg` flag (and its environment-variable fallback) exists but is rejected by those commands.

### What's missing

- **Lines 1001–4015** (75% of the file) are never read; this is exactly where `main()`, `build_parser()`/`_build_parser()`, the top-level `ArgumentParser(...)`, the `add_subparsers(...)`, per-subcommand `add_parser(...)`, and the hundreds of `add_argument(...)` calls must live.
- The definition of `_backend_kwargs` (which converts `--db` / `--pg` into API kwargs) is not visible.
- The definition of `_parse_visible_to` (used by `cmd_status`, `cmd_show`, `cmd_explain`, `cmd_trace`, `cmd_trace_access_tags`) is not visible.
- The `if __name__ == "__main__"` block, the version-string wiring (`_pkg_version`), and any `set_defaults(func=...)` bindings are not visible.

Because of this, **the search task as stated cannot be completed from these observations** — the full `add_argument` / `add_parser` set cannot be verified. The zero-match grep result should be treated as a tool malfunction and re-run (e.g. `grep -n "add_argument\|add_parser\|add_subparsers\|ArgumentParser\|set_defaults" reasons/cli.py` over the full file, or read lines 1001–4015).

## Topics to Explore

- [file] `reasons/cli.py` (lines 1001–4015) — Contains the unfetched parser construction: `ArgumentParser`, `add_subparsers`, all `add_parser` calls, and the `add_argument` surface for every subcommand; required to complete the flag audit.
- [function] `reasons/cli.py:_backend_kwargs` — Central helper that translates `--db`/`--pg` argparse values into backend kwargs, and likely handles `REASONS_PG_CONNINFO`; defines global connection-flag semantics.
- [function] `reasons/cli.py:_parse_visible_to` — Parses `--visible-to` into the `visible_to` argument for `cmd_status` / `cmd_show` / `cmd_explain` / `cmd_trace`; governs access-control filtering at the CLI boundary.
- [file] `tests/test_cli.py` — Enumerates every supported subcommand and flag used in practice (`--force`, `--any`, `--access-tags`, `--visible-to`, `--format`, `--output`, etc.); a behavior-level spec of the parser.
- [general] `cli-grep-verification` — The zero-match `find_add_argument_calls` result contradicts the argparse import and the handler reads; the search scope, glob, or regex likely needs to be corrected and re-run over the full 4015-line file.

## Beliefs

- `argparse-is-imported-in-cli` — `reasons/cli.py` imports `argparse` in its header (visible in the first-half read), so parser definitions must exist somewhere in the file.
- `cli-handlers-consume-args-not-parse-args` — Every `cmd_*` function in `reasons/cli.py` reads attributes such as `args.force`, `args.node_id`, `args.sl`, `args.any` via `getattr`, and none of them call `add_argument`; parser construction is separated from command execution.
- `add-argument-grep-result-is-anomalous` — The `find_add_argument_calls` and `find_subparser_and_parser_setup` queries returned 0 matches, which directly contradicts the presence of `import argparse` and the dozens of pre-parsed `args.*` attributes in lines 1–1000, so the search tooling, pattern, or scope is the likely fault.
- `pg-flag-must-be-rejected-explicitly` — `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, and `cmd_classify_defeaters` all short-circuit with an error when `args.pg` is set or `REASONS_PG_CONNINFO` is present, meaning `--pg` is a global flag whose validity depends on the subcommand.
- `add-command-exposes-any-mode` — `cmd_add` and `cmd_add_justification` both pass `any_mode=getattr(args, "any", False)` into the API and use `_warn_multi_premise` when `--any` is not set for 3+ premise SLs, so `--any` is a shared flag on both commands.
- `backend-selection-is-deferred-to-helper` — All handlers route persistence selection through `_backend_kwargs(args)` rather than reading `args.db`/`args.pg` directly, so the exact `--db`/`--pg` argparse definitions and `REASONS_PG_CONNINFO` fallback are implemented once in an unseen helper.
