---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Trace `write_contradiction_plan` accept handling versus `add_nogood` to verify whether plans require human acceptance.

## Summary

`write_contradiction_plan` does **not** itself accept/apply a contradiction plan. It writes a plan document and includes the command a human should run to accept it. The explicit accept path is in the CLI: `reasons/cli.py:2971-2980` parses the accepted plan and calls `apply_contradiction_plan`, which then calls `add_nogood` per entry (`reasons/api.py:4913`). By contrast, `add_nogood` is a direct mutation API: `reasons/api.py:1759` dispatches to the backend or calls `net.add_nogood(node_ids)` at `reasons/api.py:1778`.

So the plan workflow is human-in-the-loop by separation of steps, but it is not a cryptographic or status-gated approval inside `add_nogood`. Programmatic callers can call `apply_contradiction_plan` or `add_nogood` directly.

## Trace

### 1. Plan creation: `write_contradiction_plan`

- Defined at `reasons/api.py:4836`.
- The docstring notes the format is parseable by `parse_contradiction_plan()` (`reasons/api.py:4840`).
- It writes the acceptance command into the plan output:
  - `reasons/api.py:4851`: `f.write(f"  reasons contradictions --accept {output_path}\n\n")`
- The contradiction detection flow calls it from `reasons/contradictions.py:117-118`:
  - imports `write_contradiction_plan`
  - calls `write_contradiction_plan(results, output_path, append=header_written)`

This indicates `write_contradiction_plan` produces a proposal/plan file. No `add_nogood` call is visible in the observed definition body, and the generated instructions explicitly tell the user to run `--accept`.

### 2. Accept handling: CLI and `apply_contradiction_plan`

The accept path is visible in `reasons/cli.py`:

- `reasons/cli.py:2971`: `if args.accept:`
- `reasons/cli.py:2972`: `accept_path = Path(args.accept)`
- `reasons/cli.py:2976`: `plan = api.parse_contradiction_plan(accept_path.read_text())`
- `reasons/cli.py:2980`: `result = api.apply_contradiction_plan(plan, db_path=args.db)`

`apply_contradiction_plan` is defined at `reasons/api.py:4897`, and its key bridge to mutation is:

- `reasons/api.py:4913`: `result = add_nogood(entry["claims"], db_path=db_path)`

Thus accepting a contradiction plan means: read the plan file, parse it, then apply each entry by calling `add_nogood`.

### 3. Direct mutation: `add_nogood`

`add_nogood` is defined at `reasons/api.py:1759`:

- signature: `def add_nogood(node_ids: list[str], db_path: str = DEFAULT_DB, ...)`
- returns a result including `nogood_id`, `nodes`, `changed`, and `backtracked_to` (`reasons/api.py:1763`)
- dispatches to PG backend via `_pg_dispatch(...)` at `reasons/api.py:1766`
- directly calls `net.add_nogood(node_ids)` at `reasons/api.py:1778`

It has no visible `accept` or `pending` status parameter. It is an immediate state-changing operation. The CLI command at `reasons/cli.py:797` and MCP tool at `reasons/mcp_server.py:232` call `api.add_nogood(...)` directly.

### 4. Conclusion

For the contradiction-plan workflow:

- `write_contradiction_plan` writes a plan and tells the user to run `reasons contradictions --accept <path>`.
- The actual acceptance/application is a separate CLI action.
- `apply_contradiction_plan` is the accept handler and calls `add_nogood` for each accepted plan entry.
- `add_nogood` itself is not human-gated; it mutates immediately when called.

So yes, **plans require a separate human acceptance step in the normal workflow**, but that is a workflow convention, not an internal approval check in `add_nogood`.

One caveat: the observations include `reasons/api.py:4815` calling `add_nogood(c["claims"], db_path=db_path)` and a test named `test_auto_apply_calls_add_nogood` at `tests/test_contradictions.py:298`. That suggests there is also an auto-apply path for contradictions somewhere outside the visible `write_contradiction_plan` body. The observations are insufficient to confirm exactly how that path is gated.

## Topics to Explore

- [function] `reasons/api.py:write_contradiction_plan` — Inspect the full function body to verify there is no hidden call to `add_nogood` or auto-application logic.
- [function] `reasons/api.py:apply_contradiction_plan` — Confirm exactly how parsed plan entries are filtered, transformed, and applied via `add_nogood`.
- [function] `reasons/api.py:parse_contradiction_plan` — Understand the plan format, including how deleted or edited entries are represented before acceptance.
- [file] `reasons/cli.py` — Trace `--accept` argument wiring for contradictions and compare it with `cmd_accept` and `cmd_accept_proposal`.
- [general] `auto-apply-contradictions` — Reconcile `tests/test_contradictions.py:298` and `reasons/api.py:4815` with the manual `write_contradiction_plan` → `--accept` workflow.

## Beliefs

- `write-contradiction-plan-produces-plan-not-nogood` — `write_contradiction_plan` at `reasons/api.py:4836` writes a parseable plan and emits the `reasons contradictions --accept <path>` instruction at `reasons/api.py:4851`.
- `apply-contradiction-plan-calls-add-nogood` — `apply_contradiction_plan` at `reasons/api.py:4897` calls `add_nogood(entry["claims"], ...)` at `reasons/api.py:4913`.
- `contradiction-accept-requires-explicit-cli-path` — The CLI applies a contradiction plan only when `args.accept` is set, as shown at `reasons/cli.py:2971-2980`.
- `add-nogood-immediately-mutates-network` — `add_nogood` at `reasons/api.py:1759` directly invokes `net.add_nogood(node_ids)` at `reasons/api.py:1778` and has no visible acceptance-status parameter.
- `auto-apply-path-exists-outside-write-plan` — `tests/test_contradictions.py:298` and `reasons/api.py:4815` indicate an auto-apply path for contradictions distinct from the visible `write_contradiction_plan` workflow.
