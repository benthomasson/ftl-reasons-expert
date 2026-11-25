---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Pattern of resetting process-global trackers at command/test boundaries

# Pattern: Resetting Process-Global Trackers at Command/Test Boundaries

## The state that needs resetting

`reasons/llm.py` holds a module-level mutable dictionary as a cost accumulator:

- **`_cost_tracker`** — declared at `reasons/llm.py:59` with keys `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`.
- **`reset_cost_tracker()`** — defined at `reasons/llm.py:69`, and its body (lines 72–75) simply zeroes each key in-place.

Because it is a module-level object, Python keeps it alive for the entire lifetime of the interpreter process. Any code that calls the LLM increments it, and any code that reads *the summary* sees the *cumulative* total since the last reset. In a CLI that can invoke many commands in a single process (or a test run that executes many tests in one interpreter), that accumulation is exactly the wrong semantics.

## The reset pattern

`reset_cost_tracker()` is invoked **explicitly, at the start of each execution boundary**, in two places:

### 1. CLI command handlers — reset-before-each-command

`reasons/cli.py` calls it at the top of many `cmd_*` functions. Each site is a two-line idiom — an inline import plus a call:

```
2144:  from .llm import reset_cost_tracker
2145:  reset_cost_tracker()
```

The observed reset sites are:

| Command site (line) | Imports paired |
|---|---|
| `reasons/cli.py:1666-1667` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2144-2145` | `reset_cost_tracker` |
| `reasons/cli.py:2180-2181` | `reset_cost_tracker` |
| `reasons/cli.py:2213-2214` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2324-2325` | `reset_cost_tracker` |
| `reasons/cli.py:2353-2354` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2514-2515` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2581-2582` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2693-2694` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2779-2780` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2920-2921` | `reset_cost_tracker`, `format_cost_summary` |
| `reasons/cli.py:2993-2994` | `reset_cost_tracker`, `format_cost_summary` |

Two meaningful sub-patterns emerge:

- **Commands that only reset** (e.g. 2144, 2180, 2324) likely invoke the LLM but don't surface a cost report — or delegate printing elsewhere.
- **Commands that reset *and* import `format_cost_summary`** (1666, 2213, 2353, 2514, 2581, 2693, 2779, 2920, 2993) reset the tracker immediately before running their work, then presumably call `format_cost_summary()` afterwards to print a report that reflects **only this command's cost**, not the cost of every prior command the user ran in the same shell process.

### 2. Tests — reset-before-each-test

`tests/test_llm.py` imports `reset_cost_tracker` (line 19) and calls it manually at lines **345, 365, 382, 395**, i.e. at the start of individual tests that assert on tracker values. Since `tests/conftest.py` defines no autouse fixture for tracker reset (it only provides `pg_api` and skip markers), each test is responsible for resetting itself.

## Why reset *before* rather than *after*

Resetting at the *start* of each boundary is deliberately defensive:

- **Idempotent / crash-tolerant.** If a prior command raised before printing a summary, it still left the tracker dirty. A reset-at-end would never fire; a reset-at-start guarantees the next command sees a clean slate.
- **Symmetric across boundaries.** The same `reset_cost_tracker()` call works for both "beginning of a CLI command" and "beginning of a test" — no teardown hook required.
- **Zero risk of clobbering state that another caller still needs.** Because the reset is always the first thing that runs, nothing has observed the old values yet.

## Trade-offs visible in the code

- **Duplication.** The pattern is copy-pasted ~12 times in `reasons/cli.py`. A decorator (e.g. `@with_cost_tracking`) or a single reset inside the dispatcher/`main()` would express the same invariant with less surface area and avoid forgetting it on a new command.
- **No conftest-level safety net.** Tests must remember to call `reset_cost_tracker()` themselves (`tests/test_llm.py:345,365,382,395`). An `autouse=True` fixture in `tests/conftest.py` would make test isolation automatic.
- **Inline imports** at every call site (`from .llm import ...`) suggest the tracker is deliberately kept off the module's top-level import graph — probably to avoid loading LLM dependencies for commands that don't need them.

## What's missing from the observations

- The body of the CLI dispatcher / `main()` was not shown — it's possible (but not evidenced) that a centralized reset also exists there.
- The bodies of `cmd_*` functions around lines 1666–2994 aren't shown, so we can't confirm that every one of them actually calls `format_cost_summary()` after the reset. The import pairing strongly implies it.
- `format_cost_summary` is imported in `reasons/cli.py` but its definition in `reasons/llm.py` was not among the observations.
- It is not shown whether other process-global trackers exist (e.g. `_langfuse_handler` at `reasons/llm.py:158`, `_db` in `reasons/mcp_server.py:48/593`) that might need analogous reset handling.

## Topics to Explore

- [function] `reasons/llm.py:reset_cost_tracker` — See the full reset body (lines 69–75) to confirm it only zeroes the four keys and never rebinds the dict (so any aliases held elsewhere stay in sync).
- [file] `tests/test_llm.py` — Inspect which tests call the reset (lines 345, 365, 382, 395) and whether an autouse fixture would be cleaner than per-test calls.
- [file] `tests/conftest.py` — Confirm the absence of an autouse tracker-reset fixture and consider adding one to centralize test isolation.
- [function] `reasons/cli.py:main` — Look for a command dispatcher where a single `reset_cost_tracker()` could replace the 12 duplicated call sites.
- [general] `process-global-state-initialization` — Audit other module-level mutables (`_cost_tracker`, `_langfuse_handler` at `reasons/llm.py:158`, `_db` at `reasons/mcp_server.py:48/593`) to see whether they follow the same reset discipline.

## Beliefs

- `cost-tracker-is-process-global-state` — `_cost_tracker` in `reasons/llm.py:59` is a module-level dict, so its values persist across all commands and tests in a single Python process until explicitly reset.
- `reset-cost-tracker-mutates-in-place` — `reset_cost_tracker()` (lines 72–75) zeroes each key of `_cost_tracker` rather than rebinding, preserving object identity for any held references.
- `cli-requires-reset-before-each-command` — Every observed `cmd_*` reset site in `reasons/cli.py` calls `reset_cost_tracker()` at the *start* of the handler, so a subsequent `format_cost_summary()` reports only that command's cost.
- `tests-require-manual-reset` — `tests/test_llm.py` calls `reset_cost_tracker()` in individual tests (lines 345, 365, 382, 395) because `tests/conftest.py` provides no autouse reset fixture.
- `reset-before-boundary-enables-crash-safety` — Resetting at the start (rather than end) of a command/test ensures a clean slate even when the previous invocation exited without cleanup.
