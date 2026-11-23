---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _warn_multi_premise in reasons/cli.py

## `_warn_multi_premise` — CLI tip helper

### Purpose
Prints a two-line informational tip to the terminal when a user has just created an SL justification (via `reasons add` or `reasons add-justification`) that joins **3 or more** premises with AND semantics and did **not** pass `--any`. Its job is purely educational: nudge the user toward `--any` (which expands the SL into one justification per premise, giving OR semantics) in the common case where any single premise is sufficient. It's the counterpart to `_print_restoration_hints` for the reverse case (a multi-premise SL going OUT).

### Contract
- **Preconditions (assumed, not enforced):**
  - `premise_count` is an int-like value comparable with `3`.
  - `any_mode` is truthy/falsy.
  - Caller is a CLI command that already produced user-facing output, so appending stdout text is acceptable ordering-wise.
- **Postcondition:** If `premise_count >= 3 and not any_mode`, exactly two lines are written to stdout; otherwise nothing happens.
- **Invariants:** No return value, no mutation, no exception handling; the function is a pure "at most two `print`s" helper.

### Parameters
| Name | Meaning | Expected type | Edge cases |
|---|---|---|---|
| `premise_count` | Number of antecedents in the SL justification that was just created. | `int` (usually the `premise_count` field of the API result). | Callers pass `result.get("premise_count", 0)`. If the API omits the key, `0` is used and the tip is skipped (intended: "not applicable"). If the API result contains `None` for the key, `.get(..., 0)` still returns `None`, and `None >= 3` raises `TypeError`. Nothing in the type system prevents that. |
| `any_mode` | Whether the user passed `--any` (i.e., the SL should have been expanded into one justification per premise / OR-style). | `bool` (callers use `getattr(args, "any", False)`). | Truthiness is used; any truthy value suppresses the tip. |

### Return value
`None`. Callers do not consume any value; the effect is entirely the printed output.

### Algorithm
1. Evaluate `premise_count >= 3`. If false, return silently.
2. Evaluate `not any_mode`. If `--any` was used, return silently (the user already opted into OR semantics, so the tip is meaningless).
3. Otherwise `print` two lines:
   - `  Tip: This SL requires ALL {premise_count} premises to be IN. If any single`
   - `  premise is sufficient, use --any to create separate justifications.`
   
   The two-space indent is deliberate: it visually nests under the caller's preceding `Added …` / `Added justification to …` line.

### Side effects
- Writes two lines to **stdout** via builtin `print` (no `file=` argument, so **not** stderr — this is user-facing output, not an error).
- No state mutation, no I/O, no API calls, no logging.

### Error handling
- No `try`/`except`. If `premise_count` is not comparable to an `int` (e.g., `None`), a `TypeError` propagates.
- The surrounding callers catch `ValueError` (in `cmd_add`) and `(KeyError, ValueError)` (in `cmd_add_justification`), so a `TypeError` from this helper would **not** be caught and would crash the CLI. In practice this doesn't happen because the API is expected to return an `int`, but that is an assumption of the code, not a guarantee.
- When the tip's guard is false, the function swallows nothing — it simply returns.

### Usage patterns
It is called at the end of two command handlers, after a successful add:

- `cmd_add(...)` calls `_warn_multi_premise(result.get("premise_count", 0), getattr(args, "any", False))` — only after `api.add_node` succeeds and the "Added …" line is printed.
- `cmd_add_justification(...)` calls `_warn_multi_premise(result.get("premise_count", 0), getattr(args, "any", False))` — after `api.add_justification` succeeds and cascade output is printed.

**Caller obligations:**
- Pass the *SL* premise count (not CP count, not total antecedents). The name and docstring presume the caller is picking the SL-specific number.
- Pass the user's actual `--any` flag, not the API's normalized form, otherwise the tip could print when `--any` was used.
- Accept that the tip is contextual to *this one justification* — a node with several SL justifications will fire the tip once per add that itself has 3+ premises.

### Dependencies
- Only the builtin `print`. No imports, no module-level state.
- Relies conceptually on the API result containing a `premise_count` field for `api.add_node` / `api.add_justification` (see `reasons/api.py`), and on the CLI's `--any` flag being threaded through `args`.

---

## Topics to Explore
- [function] `reasons/cli.py:cmd_add` — upstream call site; shows where `premise_count` comes from in the API result and how `--any` is parsed.
- [function] `reasons/cli.py:cmd_add_justification` — the other call site; illustrates the tip firing when adding an SL to an existing node.
- [function] `reasons/cli.py:_print_restoration_hints` — conceptually similar "tips" helper for the case where a 3+ premise SL goes OUT because some premises fail, suggesting `--sl … --any` as a fix.
- [general] `--any-mode-semantics` — how `--any` expands an SL into one justification per premise in `api.add_node` / `api.add_justification`, and why that changes AND to OR.
- [general] `cli-tip-output-conventions` — the established convention of two-space-indented `Tip:`/`Note:` lines on stdout in this CLI, and whether they ever overlap with structured output.

## Beliefs
- `_warn_multi_premise-requires-premise-count-comparable-to-int` — The helper uses `premise_count >= 3` with no type coercion; passing `None` raises `TypeError` that the surrounding `except (KeyError, ValueError)` blocks do not catch.
- `_warn_multi_premise-suppresses-tip-when-any-mode` — The tip is printed only when `premise_count >= 3` **and** `any_mode` is falsy, so `--any` runs never emit it.
- `_warn_multi_premise-writes-to-stdout` — Both output lines use `print` without `file=sys.stderr`, so they interleave with normal command output rather than stderr diagnostics.
- `_warn_multi_premise-is-called-from-add-and-add-justification` — Its only call sites are `cmd_add` and `cmd_add_justification`, both after a successful API call.
- `_warn_multi_premise-encodes-and-vs-or-hint` — The message maps directly to the `--any` flag, nudging users from AND-style SL justifications (premise-set requires ALL) to OR-style ones (any single premise suffices).
