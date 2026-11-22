---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: parse_contradiction_plan in reasons/api.py

# `parse_contradiction_plan` — `reasons/api.py`

## Purpose

`parse_contradiction_plan` is the read-side of a snapshot-based review workflow: `write_contradiction_plan` writes a markdown plan whose headers look like `### NOGOOD <id> [APPLY]`, a human edits the file (flipping `[APPLY]` to `[SKIP]` or deleting entries), and this function re-reads that file into a plain, structured list that `apply_contradiction_plan` can turn into `add_nogood(...)` calls.

It exists to decouple the markdown plan file from the calling code: `apply_contradiction_plan` never has to parse prose. It only sees `[{"id": ..., "claims": [...]}]`.

## Contract

**Preconditions (destroyed on violation, not enforced by type system):**
- `plan_text` is a *string* (anything `.splitlines()` supports; `None` will raise `AttributeError`).
- The file *format* matches `write_contradiction_plan` — headers exactly `### NOGOOD <id> [APPLY|SKIP]` (three hashes, `APPLY`/`SKIP` uppercase), claims lines start with `- Claims:`.
- Claim IDs do **not** contain commas (they are split on `,` naively).

**Postconditions:**
- Returns a `list[dict]` where every dict has keys `"id"` (str) and `"claims"` (non-empty `list[str]`).
- Only `[APPLY]` entries appear; `[SKIP]` entries and entries that never got a `- Claims:` line are silently absent.
- Order of the returned entries matches the order in which the `### NOGOOD ...` headers appear in the file.
- No deduplication, no ordering of claims, no semantic validation.

**Invariants maintained during parsing:**
- `current_id is not None` ⇔ the parser is "inside" an `[APPLY]` block (or immediately after such a block, before the next header).
- `current_claims` is reset to `[]` at every header, so a stale claims list from a previous block can never leak into the next.

## Parameters

| name | type | meaning | edge cases |
|---|---|---|---|
| `plan_text` | `str` | Full contents of a `proposed-contradictions.md` file produced by `write_contradiction_plan` | Empty string → returns `[]`. Non-string → `AttributeError` from `.splitlines()`. CRLF line endings are fine because `splitlines()` normalizes. |

Note: the type hint is `str`, but there is no runtime check. Passing bytes will fail deep inside `re.match` or `.splitlines()`.

## Return Value

`list[dict]`, each element shaped `{"id": str, "claims": list[str]}`.

- `id` is `m.group(1)` from the header regex — never empty and never contains whitespace, thanks to `\S+`.
- `claims` items are whitespace-stripped and with *leading/trailing backticks removed*, but they may still contain spaces (e.g. `"foo bar"` from `- Claims: foo bar, baz` — no comma inside so parsed as one claim). Backtick stripping only strips from the ends, not inner backticks.
- Returned list can be empty.
- The caller (`apply_contradiction_plan`) is responsible for handling failures inside `add_nogood` — this function will happily produce entries referring to node IDs that don't exist in the DB.

## Algorithm

Let `S` = stream of lines from `plan_text.splitlines()`.

1. Initialize `entries = []`, `current_id = None`, `current_claims = []`.
2. For each `line` in `S`:
   - **Header test:** `re.match(r"###\s+NOGOOD\s+(\S+)\s+\[(APPLY|SKIP)\]", line)`.
     - `re.match` only anchors at `^`, so the regex will match a line like `### NOGOOD x [APPLY] ...` (trailing junk after the bracket is allowed).
     - If it matches:
       - **Flush** the currently accumulating block: if `current_id` is truthy **and** `current_claims` is non-empty, append `{"id": current_id, "claims": current_claims}`.
       - Set `current_id = m.group(1)` if `m.group(2) == "APPLY"`, otherwise `None` (this actively disables subsequent Claims capture for `SKIP`).
       - Reset `current_claims = []` (unconditional, even when switching to `SKIP`).
       - `continue` (nothing else to parse on this line).
   - **Claims test:** if `current_id` is truthy **and** `line.strip().startswith("- Claims:")`:
     - Take everything after the first `:` — `line.split(":", 1)[1]` — and `.strip()` it.
     - Split on `,`, `.strip()` each piece, then `.strip("`")` to strip outer backticks.
     - Filter out empty pieces (`if c.strip()`).
     - **Assign** (not append) to `current_claims`, so a second `- Claims:` line inside the same entry overwrites the first. (This is a subtle behavior — a reviewer won't normally produce two, but it is not enforced.)
3. After the loop: **flushing again** is necessary because the final entry has no trailing header to trigger a flush.
   - Same `if current_id and current_claims` guard.

Two consequences worth noting:

- A header immediately followed by another header will discard the first (its `current_claims` is empty at flush time — nothing appended). This is intentional: the format only advances state on the next header, and the guard `current_id and current_claims` skips partially-filled blocks.
- Any line that is neither a header nor a `- Claims:` line while `current_id` is set is **ignored** (e.g. `- Severity: ...`, `- Analysis: ...`). This matches the writer, which emits those extra lines.

## Side Effects

None. `parse_contradiction_plan` is a **pure function**:

- No I/O, no logging, no network access, no DB access.
- No mutation of the input (strings are immutable; only local variables change).
- Safe to call concurrently.

## Error Handling

There is **no explicit error handling** — no `try/except`, no validation of `plan_text`. Consequently:

| situation | behavior |
|---|---|
| `plan_text` is `None` | `AttributeError: 'NoneType' object has no attribute 'splitlines'` |
| `plan_text` is bytes | `TypeError` from `splitlines()` returning bytes or from `re` |
| Malformed header (e.g. `[ apply ]`) | Line is silently ignored — no entry, no error |
| `### NOGOOD x [APPLY]` with no `- Claims:` line | Silently dropped |
| Claims contain embedded commas | Naive `split(",")` breaks the claim into multiple pieces |
| An `[APPLY]` entry uses lowercase `apply` | Regex is case-sensitive → silently dropped |

The function therefore assumes the plan file was machine-generated by `write_contradiction_plan` and only edited in the *header marker* (or deleted wholes), not overwritten with arbitrary markdown.

## Usage Patterns

The canonical pipeline, wired up by the CLI:

```python
contradictions = detect_contradictions(db_path=db_path, ...)
write_contradiction_plan(contradictions, "proposed-contradictions.md")
# human edits file, flips APPLY -> SKIP
plan_text = Path("proposed-contradictions.md").read_text()
plan = parse_contradiction_plan(plan_text)
result = apply_contradiction_plan(plan, db_path=db_path)
```

Caller obligations:

- Must repair checks (e.g. verify referenced node IDs exist) — this function performs none.
- Should tolerate the fact that a header written as `### NOGOOD foo [APPLY]` **without** a `- Claims:` line contributes nothing.
- Should not assume that ID casing/whitespace was normalized — `\S+` preserves whatever the writer put.

## Dependencies

- Only `re` from the standard library (imported locally inside the function, despite the module already importing `re` at the top — a redundant but harmless local import).
- Logically coupled to `write_contradiction_plan` (the two are an inverse pair) and to `apply_contradiction_plan` (its consumer).

## Assumptions the Type System Does Not Enforce

1. **Format coupling** — the regex literally encodes the output format of `write_contradiction_plan`; if the writer ever changes `###` to `##` or adds a leading marker (e.g. `<!-- -->`), this silently returns nothing.
2. **Case sensitivity** — `APPLY`/`SKIP` are matched exactly.
3. **No comma in claims** — a claim like `"foo, bar"` (as written by `write_contradiction_plan` from `', '.join(...)`) would be split into two claims.
4. **One `- Claims:` line per block** — a second one silently replaces the first.
5. **IDs are unique** — nothing dedupes duplicate `### NOGOOD` headers; both entries will appear in the result and each will attempt `add_nogood` independently.
6. **`current_claims` should be non-empty** — only enforced by the `and current_claims` guard at flush; entries with no claims simply drop out, which is inconsistent with "actionable entries" semantics if a reviewer expected every `[APPLY]` to be applied.

## Topics to Explore

- [function] `reasons/api.py:write_contradiction_plan` — The producer whose output this function parses; read together they form the plan format contract, including the exact `### NOGOOD <id> [APPLY]` header and `- Claims:` line.
- [function] `reasons/api.py:apply_contradiction_plan` — The consumer that turns parsed entries into `add_nogood` calls and this function's return value must match; it defines what "actionable" means downstream.
- [function] `reasons/api.py:detect_contradictions` — Where the contradictions that end up in the plan file originate; useful to see how `claims` are computed.
- [function] `tests/test_contradictions.py` — Likely contains round-trip tests (write → edit → parse) and edge cases for `[SKIP]` and same-line-multiple-claims.
- [general] `plan-file-formats` — The general design pattern of human-editable markdown ceremonies in this codebase (`parse_dedup_plan` / `write_dedup_plan` follow the same shape).

## Beliefs

- `parse-contradiction-plan-is-pure` — `parse_contradiction_plan` performs no I/O, no DB access, and mutates no shared state; it is safe to call repeatedly or concurrently.
- `apply-marker-drives-inclusion` — Only headers ending in `[APPLY]` (case-sensitive) contribute entries; `[SKIP]` and any other casing are ignored, and a `SKIP` block's claims lines are also ignored.
- `entries-require-claims` — An `[APPLY]` header without a subsequent `- Claims:` line is silently dropped from the result, so the parser cannot express "apply with empty claims".
- `claims-split-on-comma` — Claim IDs are split on literal `,` and stripped of outer backticks; embedded commas inside a claim are not supported.
- `header-flushes-previous-block` — The parser is streaming: the previous block is flushed when the next `### NOGOOD` header is seen, and the final block is flushed after the loop.
