---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore how the CLI surfaces mixed success/error tuples and whether retries or rollback are expected.

## Summary

**The CLI does not use mixed success/error tuples.** The pattern throughout `reasons/cli.py` is:

- **API functions return a plain `dict` on success** (with fields like `node_id`, `truth_value`, `changed`, `cascade`, `old_truth_value`, `new_truth_value`, `domain_id`).
- **Failures are raised as exceptions** (`FileExistsError`, `ValueError`, `KeyError`, `IndexError`) and translated by the CLI into `Error: ...` on **stderr** plus `sys.exit(1)`.
- **Idempotent "no-op" outcomes** (`already OUT`, `already IN`) are reported on **stdout** and exit **0** — they are treated as successful invocations, not errors.
- **Neither retries nor rollback are surfaced, attempted, or expected by the CLI.** No `retry|rollback|commit|transaction|savepoint|atomic|abort` hits appear in any file the search covered. Each CLI invocation is treated as a single atomic-from-the-user's-view step: either print success (exit 0) or print error (exit 1).

## What the CLI code actually does

### 1. Result dicts, not tuples
Every command in `reasons/cli.py` follows the same shape, e.g. `cmd_add` (~line 35) and `cmd_add_justification` (~line 60):

```python
try:
    result = api.add_node(...)          # or add_justification, retract_node, ...
    print(f"Added {result['node_id']} [{result['truth_value']}] ({result['type']})")
except ValueError as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

There is no `(success, error)` or `(results, failures)` return value anywhere in the visible CLI surface. The only "shape" distinction is **`result.get("changed")`** in `cmd_retract`, `cmd_remove_justification`, `cmd_mark_duplicate`, `cmd_defeat_justification`, and `cmd_mark_superseded` — but that's a **payload field describing cascade effects**, not an error signal (e.g., a retraction of an already-OUT node returns `changed=False` and exits 0).

### 2. `cmd_init` branches on result shape
`cmd_init` (~line 17) does the one notable "two-form success" split:

```python
if "db_path" in result:
    print(f"Initialized RMS database: {result['db_path']}")
else:
    print(f"Initialized PostgreSQL domain: {result['domain_id']}")
```

This is a **backend-discriminating key**, not a mixed-status tuple. Both branches are success.

### 3. Idempotent success is explicit
`cmd_retract` (~line 133) and the corresponding `assert`/`mark-*` paths surface "nothing to do" as a normal outcome:

```python
if not result["changed"]:
    print(f"{args.node_id} is already OUT")
else:
    print(f"Retracted {args.node_id}")
    _print_cascade(result)
    ...
```

`tests/test_cli.py` pins this: `test_retract_already_out` and `test_assert_already_in` both assert `code == 0` (the 0-valued outcome of `run_cli` when `main()` does not raise `SystemExit`, plus all `assert code == 0` checks). Conversely `test_add_duplicate_fails` asserts `code == 1`. So the test suite treats **no-op as success** and **rule violation as failure** — never a mixed tuple.

### 4. Backend-gated commands bail early
`cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, and `cmd_migrate_defeaters` all start with:

```python
if getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO"):
    print("Error: ... not supported with --pg ...", file=sys.stderr)
    sys.exit(1)
```

That's a pre-flight rejection, not a runtime mixed result. There's no partial outcome or retry attempt.

### 5. Import modules use counters, not error tuples
`reasons/import_beliefs.py:import_into_network` tracks `imported`, `skipped`, `retracted` counters and returns a **summary dict** (starts at the bottom of the visible 250 lines). `reasons/import_agent.py` uses `_import_nogoods` returning `count`, and `_update_node_justifications` replaces `justifications` then calls `network._rebuild_dependents()` — direct mutation of in-memory state. So the "mixed-ness" in the import path is expressed as **numeric skip/retract counts**, not as a list of error tuples or a retry queue.

## What is absent: retries and rollback

- **No retries.** No `for attempt in range`, no backoff, no `--retry` args, no retry parameters passed into `api.*`. The error handlers all terminate with `sys.exit(1)`.
- **No rollback / transaction primitives.** The dedicated search for `rollback|retry|commit|transaction|savepoint|atomic|abort` returned `total_count: 0` across the searched files. The import modules mutate the `Network` in place via Python objects (e.g. `network.nodes[node_id].justifications = new_justifications` in `reasons/import_agent.py:_update_node_justifications`) without a visible begin/savepoint/commit boundary in the observed lines.
- **Implied assumption:** the API layer (`reasons/api.py`, not shown) is expected to be all-or-nothing per call. From the CLI's perspective, a raised exception means the call did not take effect — which is only true if the underlying `SqliteApi`/`PgApi` methods wrap mutations in their own transaction. That contract is **not verifiable from the provided observations**.

## Data caveats worth flagging

The `search_cli_output_handling` and `search_cli_tests_expectations` results reported `total_count: 0` for a pattern that includes `result` and `error` — which clearly appear dozens of times in the shown CLI code. This is **internally inconsistent**: either the search was scoped to a limited line range, targeted a different working tree, or the search tool failed silently on these files. I would not treat those zero-hit results as evidence of absence; the negative finding on `rollback|retry|commit|transaction|savepoint|atomic|abort` is more credible because it's supported by direct inspection of the shown functions, but it **still only covers the first ~300 lines of `cli.py` and the first ~250 lines each of `import_agent.py` / `import_beliefs.py`**.

## Topics to Explore

- [file] `reasons/api.py` — The unseen layer that determines whether errors are raised atomically or whether partial mutations are committed; this decides whether the CLI's "exit 1 means no change" assumption holds.
- [file] `reasons/cli.py` — Only lines 1–300 were observed; the `main()` dispatcher, `_backend_kwargs`, and any batch/force commands after line 300 may harbour retry-batch or mixed-result logic.
- [function] `reasons/db.py:SqliteApi.add_node` — The concrete transaction/commit boundary for a single CLI mutation; needed to verify rollback semantics.
- [file] `reasons/import_agent.py` — Lines 250–600 are unobserved; the `import_*` entrypoints may have a `--dry-run`, skip-list, or partial-failure reporting path that changes the "no mixed tuples" conclusion.
- [general] `pg-vs-sqlite-parity` — Several commands explicitly reject `--pg` at runtime; understanding which operations are SQLite-only clarifies why no retry/rollback uniform contract exists across backends.

## Beliefs

- `cli-uses-exceptions-not-tuples` — Every command handler in `reasons/cli.py` (e.g. `cmd_add`, `cmd_add_justification`, `cmd_retract`) returns `None` and signals errors by raising and catching exceptions that terminate with `sys.exit(1)`; no command returns a `(result, error)` tuple.
- `cli-treats-idempotent-noop-as-success` — `cmd_retract` prints `"<id> is already OUT"` and exits 0 when `result["changed"]` is falsy, and `test_retract_already_out`/`test_assert_already_in` in `tests/test_cli.py` assert exit code 0.
- `cli-produces-error-messages-on-stderr` — All error paths in `reasons/cli.py` write `f"Error: {e}"` to `file=sys.stderr` and exit with code 1, matching `assert "Error" in err` in CLI tests.
- `cli-has-no-retry-logic` — No retry loops, backoff, or retry parameters exist in the observed CLI functions; failures flow straight to `sys.exit(1)`.
- `cli-has-no-rollback-semantics` — The observed CLI and import modules contain no `commit`, `rollback`, `transaction`, or `savepoint` calls; in-place mutations like `reasons/import_agent.py:_update_node_justifications` rely on the (unobserved) `reasons/api.py` layer for atomicity.
- `cli-gates-pg-unsupported-commands` — `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, and `cmd_migrate_defeaters` short-circuit with `sys.exit(1)` when `--pg` or `REASONS_PG_CONNINFO` is set.
