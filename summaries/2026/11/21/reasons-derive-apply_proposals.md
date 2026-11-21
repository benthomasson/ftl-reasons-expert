---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: apply_proposals in reasons/derive.py

### Purpose
`apply_proposals` is the persistence step for accepted derivations. It takes proposals that have already been parsed and validated, and attempts to insert each one into the reasons database as a derived node via `api.add_node`.

It exists to batch-apply proposals while isolating per-proposal failures: one bad proposal should not abort the whole batch. The caller gets a per-proposal result list containing either the raw `api.add_node` result or an error string.

### Contract
Preconditions:

- `valid` is an iterable of proposal dictionaries, normally the output of `validate_proposals(parse_proposals(...), nodes)`.
- Each proposal is expected to contain at least:
  - `"id"`: string node ID
  - `"text"`: string claim text
  - `"antecedents"`: iterable of string node IDs
  - `"unless"`: iterable of string node IDs, or a falsey value such as `None`/`[]`
  - `"label"`: string rationale/label
  - optional `"mode"`: expected to be `"all"` or `"any"`
- `db_path` points to a database that `api.add_node` can open/write.
- The proposals are assumed to be valid: existing antecedents, non-existing proposed ID, at least two antecedents, etc. This function does not re-run validation.

Postconditions:

- For every iterated proposal, `results` contains a tuple `(p, result_or_error_string)`, in iteration order.
- Successful proposals have been passed to `api.add_node`, causing database side effects.
- Failed proposals are represented by a string error value.
- There is no all-or-nothing guarantee.

Important invariant:

- The function always passes `source_type="derived"`.
- `any_mode` is true only if `p.get("mode") == "any"` exactly.
- `antecedents` are serialized as a comma-joined string.
- `unless` is serialized as a comma-joined string, or `""` when falsey.

### Parameters
- `valid`: iterable of proposal dicts. Expected to be prevalidated. A generator is okay because the function iterates once. If `valid` itself is not iterable, the outer `for` raises and is not caught.
- `db_path`: database path passed through to `api.add_node`. Defaults to `"reasons.db"`. The function does not validate or normalize it.

Expected proposal fields:

- `p["antecedents"]`: passed to `",".join(...)`. Should be a list/tuple/set of strings. If it is `None`, a non-string iterable, or missing, the per-proposal `try` catches the resulting error.
- `p["unless"]`: read directly with `p["unless"]`, not `p.get("unless")`. Missing key causes a per-proposal error. Falsey values become `""`.
- `p["id"]`, `p["text"]`, `p["label"]`: passed directly to `api.add_node`. Missing keys or wrong types become per-proposal errors.
- `p.get("mode")`: optional. Only exact lowercase `"any"` turns on `any_mode`. `"ANY"`, `"all"`, `"ALL"`, missing mode, or any other value maps to `False`.

### Return Value
Returns a list of tuples:

```python
[(proposal, result_or_error_string), ...]
```

For each proposal:

- On success, the second element is whatever `api.add_node(...)` returned. The docstring calls this a `result_dict`, but this function does not enforce that shape.
- On failure, the second element is `str(e)`, a plain exception message string.

The caller must handle mixed result types. A string means that proposal failed; a non-string is expected to be the `api.add_node` success result.

### Algorithm
1. Initialize `results = []`.
2. For each proposal `p` in `valid`:
   1. Join `p["antecedents"]` with commas into `sl`.
   2. If `p["unless"]` is truthy, join it with commas; otherwise use `""`.
   3. Call `api.add_node(...)` with:
      - `node_id=p["id"]`
      - `text=p["text"]`
      - `sl=sl`
      - `unless=unless`
      - `label=p["label"]`
      - `source_type="derived"`
      - `any_mode=(p.get("mode") == "any")`
      - `db_path=db_path`
   4. Append `(p, result)`.
3. If any exception occurs inside the per-proposal block, append `(p, str(e))` instead.
4. Return `results`.

### Side Effects
- Calls `api.add_node` once per proposal, which writes derived nodes and justifications to the reasons database.
- Earlier successful writes remain even if later proposals fail. This function has no batch transaction or rollback.
- Does not mutate the proposal dictionaries.
- Does not log, print, or write proposal files.
- Does not catch errors raised by iterating `valid` itself; only errors inside each proposal’s processing are caught.

### Error Handling
- Catches `Exception` per proposal. This includes:
  - `KeyError` from missing proposal fields
  - `TypeError` from joining non-string iterables
  - errors raised by `api.add_node`
  - database write failures
- Does not catch `BaseException` subclasses such as `KeyboardInterrupt` or `SystemExit`.
- Converts failures to `str(e)`, losing traceback and exception type.
- Does not re-raise per-proposal failures.
- Does not roll back previously applied proposals.
- Does not validate that `p["antecedents"]` has at least two entries; it will happily join an empty list to `""`, though `validate_proposals` normally prevents that.
- Does not validate that `p["unless"]` exists. Unlike `mode`, `unless` is required.
- Does not normalize `mode`; only exact `"any"` is treated as ANY.

### Usage Patterns
Typical pipeline:

```python
proposals = parse_proposals(response)
valid, skipped = validate_proposals(proposals, nodes)
results = apply_proposals(valid, db_path=db_path)
```

The caller should:

- Inspect every `(proposal, result)` tuple.
- Treat string results as failures and report/retry them if desired.
- Not assume atomicity; some nodes may already have been written even if later proposals fail.
- Pass the same database path used by the rest of the reasons workflow.
- Use `write_proposals_file` instead when proposals should be reviewed manually before application.

### Dependencies
- `reasons.api.add_node` is the only internal dependency used by this function.
- It relies on the proposal schema produced by `parse_proposals` and enforced by `validate_proposals`.
- It assumes `api.add_node` accepts `node_id`, `text`, `sl`, `unless`, `label`, `source_type`, `any_mode`, and `db_path`.

### Assumptions Not Enforced by the Type System
- `valid` elements are dictionaries with the expected keys.
- `antecedents` and `unless` contain strings without commas; comma-containing IDs would corrupt the `sl`/`unless` serialization.
- `mode`, when provided, is already lowercased. `"ANY"` silently becomes ALL behavior.
- `unless` exists even when empty.
- The database schema and `api.add_node` semantics match the caller’s expectations.
- The proposal ID does not already exist and antecedents are valid; this function trusts `validate_proposals`.

## Topics to Explore
- [function] `reasons/api.py:add_node` — Understand how comma-separated `sl`/`unless`, `any_mode`, `source_type`, and `db_path` are interpreted, and what the success result contains.
- [file] `reasons/derive.py` — Read `parse_proposals` and `validate_proposals` to see the proposal schema and prechecks this function relies on.
- [file] `tests/test_derive.py` — Check expected behavior for applying proposals and per-proposal error handling.
- [file] `tests/test_source_type.py` — Verify how `source_type="derived"` and derived/GATE semantics are tested.
- [general] `partial-failure-handling` — Explore how the CLI surfaces mixed success/error tuples and whether retries or rollback are expected.

## Beliefs
- `apply-proposals-catches-per-proposal-exceptions` — Any `Exception` raised while joining proposal fields or calling `api.add_node` is caught and converted to a string result for that proposal.
- `apply-proposals-requires-unless-key` — The function directly indexes `p["unless"]`, so a missing `unless` key causes a per-proposal `KeyError` result instead of defaulting to an empty list.
- `apply-proposals-maps-mode-any-only-lowercase` — `any_mode` is passed as true only when `p.get("mode")` equals the exact lowercase string `"any"`.
- `apply-proposals-sets-source-type-derived` — Every successful `api.add_node` call receives `source_type="derived"`.
- `apply-proposals-has-no-batch-rollback` — It catches failures inside the loop and does not roll back earlier successful `api.add_node` calls if a later proposal fails.
