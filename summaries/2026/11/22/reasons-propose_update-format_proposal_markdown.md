---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: format_proposal_markdown in reasons/propose_update.py

# `format_proposal_markdown` — Explanation

## Purpose

Renders a single update/retraction *proposal* (as produced by `parse_update_proposals` from an LLM) into a self-contained markdown section that a human can review. It exists to bridge the machine pipeline (`propose_updates` → `format_proposals_file`) with a human-in-the-loop review step: the reader sees the current belief text, the suggested change, the failure-mode/basis classification, any cascade side-effects, and a copy-pasteable CLI command to accept the proposal.

## Contract

- **Preconditions**
  - `proposal` is a mapping with at least `proposal["id"]` (any hashable, used as-is) and `proposal["action"]` (a string; typically `"update"` or `"retract"`).
  - `nodes`, if supplied, is a mapping `{node_id -> node_dict}` where each node_dict has a `"text"` key (or `nodes.get(nid, {})` returns `{}`).
  - `cascade`, if supplied, is a mapping with `"retracted"`, `"restored"` (lists of dicts each containing `"id"` and optionally `"text"` as a string), and `"total_affected"` (an `int`).
- **Postconditions**
  - Returns a string that begins with `## {ACTION}: {id}` followed by a blank line, then a sequence of `**Label:** value` lines and — if applicable — cascade listings and a `**Command:**` line.
  - The returned string has *no* trailing newline; callers join it with a surrounding `\n`.
  - It is pure and idempotent: the same inputs produce the same output.
- **Invariants**
  - `**Current text:**`, `**Failure mode:**`, `**Basis:**`, `**Cascade impact:**`, and either a `**Command:**` line or none are always present (the failure-mode and basis fields default to `"unknown"`).
  - A `**Proposed text:**` block and a `reasons update` command appear only when `action == "UPDATE"` **and** `proposed_text` is truthy.
  - A `reasons retract` command appears only when `action == "RETRACT"`.

## Parameters

| Param | Type | Default | Notes |
|---|---|---|---|
| `proposal` | mapping | required | Contributes `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`. Unknown/missing optional keys are read via `.get(..., default)`. |
| `nodes` | mapping or `None` | `None` | Used only to look up `node["text"]` for the `**Current text:**` line. Missing nodes silently yield an empty current text. |
| `cascade` | mapping or `None` | `None` | When `None` (or empty dict — falsy) prints `**Cascade impact:** not computed`. Otherwise reads `retracted`, `restored`, `total_affected`. |

## Return Value

A single markdown string. The caller is responsible for:
- Adding a trailing newline / blank line if desired (the aggregate builder does this).
- The fact that the header uses the uppercased action (which is what the *display* uses, whereas `proposal["action"]` itself is lowercase).

## Algorithm (step-by-step)

1. Extract `nid` and `action` (uppercased). Build `lines = ["## <ACTION>: <nid>", ""]` (the `""` guarantees a blank line after the header).
2. **Current text.** If `nodes` is truthy, look up `nodes.get(nid, {})` and read `.get("text", "")`; otherwise empty. Append `**Current text:** ...` (no leading newline — it comes from the prior `""`).
3. **Proposed text.** Only for UPDATE actions with truthy `proposed_text`. Append `\n**Proposed text:** ...`.
4. Append `\n**Failure mode:** ...` and `**Basis:** ...` — both fall back to `"unknown"`.
5. **Evidence** is appended only if truthy, with no leading `\n` (so it sticks directly under `Basis`).
6. **Cascade block** (always emits a `**Cascade impact:**` line):
   - Falsy `cascade` → `"not computed"`.
   - Truthy but `total_affected <= 0` → `"none"`.
   - `total_affected > 0`:
     - **RETRACT** header: `"{n} dependent(s) affected"`, and every retracted item gets `— will go OUT`, every restored item gets `— will go IN`.
     - **Non-RETRACT** (UPDATE) header: `"{n} dependent(s) need re-review"`; both retracted and restored items are listed with no suffix (they're all candidates for review).
   - Each item line is `- {id}: "{text[:80]}"`, i.e. text is truncated to 80 characters.
7. **Command line** (only one of):
   - UPDATE with `proposed_text`: escape `"` → `\"` only, then emit `reasons update {nid} "{escaped}"`.
   - RETRACT: emit `reasons retract {nid} --reason "{failure_mode}: {comment}"` (neither `failure_mode` nor `comment` is escaped/truncated).
8. Return `"\n".join(lines)`.

## Side Effects

- **None.** This is a pure formatting function: no I/O, no mutation of `proposal`, `nodes`, or `cascade`, no printing, no file writes.

## Error Handling

There is no `try/except`. What can go wrong:

| Failure | Where |
|---|---|
| `KeyError` | `proposal["id"]`, `proposal["action"]` if keys absent. |
| `AttributeError` | `proposal["action"].upper()` if `action` isn't a string. |
| `TypeError` | `proposal["proposed_text"].replace(...)` if `proposed_text` is truthy but not a string (e.g., a list). Also `item.get("text", "")[:80]` if `text` is present but `None` (slicing `None`). |
| `KeyError` | `item['id']` in the cascade loops if any cascade item lacks `"id"`. |

The function intentionally does *not* defend against malformed inputs — it trusts the caller to hand it the structured output of `parse_update_proposals`. Callers higher up (`format_proposals_file`, `propose_updates`) therefore treat parse-produced dicts as the contract.

## Usage Patterns

- Normally invoked only by `format_proposals_file`, which iterates the proposals, fetches `cascades.get(nid)`, wraps each section with `---` separators and blank lines.
- Can be called directly when a caller wants a single-proposal preview (e.g., tests).
- **Caller obligations**: mutate/escape both `proposed_text` and — if applicable — `comment`/`failure_mode` before relying on the printed command, because only double quotes are escaped in the `reasons update` path and *nothing* is escaped in the `reasons retract` path.

## Dependencies

- **Internal**: none imported here — it's a leaf utility.
- **Data-shape dependencies** (implicit contracts, not enforced by types):
  - `proposal`: produced by `parse_update_proposals` (keys `id`, `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, `comment`).
  - `nodes`: the TMS belief map (values with a `text` key).
  - `cascade`: the output of a cascade-prediction routine with `retracted`/`restored`/`total_affected` — the shape is *not* defined in this module. That's the most important external dependency to trace.

---

## Topics to Explore

- [function] `reasons/propose_update.py:format_proposals_file` — Shows the exact document-level structure and where `cascade=None` vs `cascade={...}` is chosen.
- [function] `reasons/propose_update.py:parse_update_proposals` — Defines the "proposal dict" contract this function assumes, including which fields are always present vs optional.
- [general] `cascade-computation` — Where `cascade["retracted"]`, `cascade["restored"]`, and `cascade["total_affected"]` are populated; not visible in this file but drives the most complex branches here.
- [file] `tests/test_propose_update.py` — Expected output for UPDATE-with-cascade, RETRACT-with-cascade, missing nodes, and `cascade=None` edge cases.
- [general] `shell-escaping-in-cli-commands` — Only `"` is escaped in the printed command; explore how `reasons update` parses arguments and whether `$`, backticks, or backslashes in `proposed_text` could break the copied command.

## Beliefs

- `format-proposal-markdown-requires-id-and-action` — The function assumes `proposal` has both `"id"` and `"action"` keys; missing either raises `KeyError` immediately.
- `action-is-uppercased-for-display-and-branching` — `proposal["action"].upper()` is compared against `"UPDATE"`/`"RETRACT"`, so actions are case-insensitive in the input.
- `cascade-none-renders-not-computed` — When `cascade` is `None` (or `{}`), the output always contains `**Cascade impact:** not computed` rather than omitting the line.
- `command-escapes-only-double-quotes` — The `reasons update` line escapes only `"` in `proposed_text`; `reasons retract` escapes nothing in `failure_mode` or `comment`.
- `cascade-item-text-truncated-to-80-chars` — Every cascade listing entry slices `item["text"]` to `[:80]`, so long belief texts are silently clipped without an ellipsis marker.
