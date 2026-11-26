---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: format_proposals_file in reasons/propose_update.py

## Purpose

`format_proposals_file` is the aggregate Markdown formatter for a list of update proposals. It produces a single review document containing a title, usage instructions, and one Markdown section per proposal. It exists to turn the structured proposal dictionaries returned by `propose_updates` / `parse_update_proposals` into a human-readable file that a reviewer can read and act on via suggested `reasons update` / `reasons retract` commands.

It delegates the per-proposal rendering to `format_proposal_markdown`.

## Contract

**Preconditions**

- `proposals` is expected to be a falsy value such as `None` or `[]`, or a sequence/iterable of proposal dictionaries.
- If `proposals` is non-empty, each proposal must at least contain an `"id"` key, because this function directly evaluates `proposal["id"]`.
- More implicitly, each proposal must also be valid input for `format_proposal_markdown`, which requires `"action"` and reads other fields such as `"proposed_text"`, `"failure_mode"`, `"basis"`, `"evidence"`, and `"comment"`.
- `nodes`, when provided, should be a mapping from node ID to node dictionary. It is passed through to `format_proposal_markdown` to render the current belief text.
- `cascades`, when truthy, should be a mapping from proposal/node ID to a cascade-impact dictionary. If falsey, no cascade lookup is attempted.

**Postconditions**

- Always returns a single `str`.
- The returned document always begins with:
  - `# Proposed Updates`
- If `proposals` is falsy, the returned document includes `No updates proposed.` and does not include proposal separators.
- If `proposals` is non-empty, each proposal is preceded by `---` and rendered by `format_proposal_markdown`.
- For non-empty proposals, the final returned string ends with a newline because the loop appends an empty line after the final proposal and then joins with `"\n"`.

**Invariants / assumptions**

- The function does not mutate `proposals`, `nodes`, or `cascades`.
- It assumes proposals are concrete sequences or otherwise falsy when empty. Passing an empty generator would not hit the `No updates proposed.` branch because an empty generator object is truthy.
- It assumes `proposal["id"]` is a valid node ID and that the optional `cascades` mapping is keyed by that same ID.
- It assumes cascades are either absent/falsey or mapping-like with a `.get` method.

## Parameters

- `proposals` — The proposals to render. Expected to be a list of dicts from `parse_update_proposals` or equivalent. Each non-empty proposal must have `"id"`; per-proposal rendering also requires `"action"`. A falsy value means “no proposals.”
- `nodes=None` — Optional mapping of node ID to node dict. Used by `format_proposal_markdown` to render `Current text`. If omitted, the current text is rendered as an empty string.
- `cascades=None` — Optional mapping of node ID/proposal ID to cascade-impact data. If truthy, the function looks up `cascades.get(nid)` for each proposal. If absent or falsey, `cascade` is `None`, and `format_proposal_markdown` renders cascade impact as `not computed`. If the mapping exists but lacks the proposal ID, `.get` returns `None`, also resulting in `not computed`.

## Return Value

Returns a Markdown document as a string.

- If `proposals` is falsy:
  - Returns the header plus `No updates proposed.`
  - No `---` separators are emitted.
- If `proposals` is non-empty:
  - Returns the header followed by one section per proposal.
  - Each proposal section is prefixed by `---` and a blank line.
  - The document ends with a trailing newline.

The caller is responsible for writing the string to a file, printing it, or otherwise presenting it. The function itself does not perform I/O.

## Algorithm

1. Initialize `lines` with the document title, an empty line, an instruction sentence, and another empty line.
2. Check `if not proposals`.
   - If true, append `"No updates proposed."` and return `"\n".join(lines)`.
3. Otherwise, iterate over each proposal:
   - Read `nid = proposal["id"]`.
   - Determine the cascade for this proposal:
     - If `cascades` is truthy, use `cascades.get(nid)`.
     - Otherwise use `None`.
   - Append `"---"`.
   - Append an empty line.
   - Append the result of `format_proposal_markdown(proposal, nodes, cascade)`.
   - Append an empty line.
4. Return `"\n".join(lines)`.

## Side Effects

None. The function is pure formatting logic:

- No file writes.
- No network calls.
- No mutations of input dictionaries or mappings.
- No global state changes.

## Error Handling

There is no explicit exception handling; errors propagate.

Potential exceptions include:

- `KeyError` if a non-empty proposal lacks `"id"` in `format_proposals_file`, or lacks `"action"` in the delegated `format_proposal_markdown`.
- `AttributeError` if `proposal["action"]` is not a string and `format_proposal_markdown` calls `.upper()`.
- `AttributeError` if a truthy `cascades` value does not have a `.get` method.
- `AttributeError` if `nodes` is provided but is not mapping-like and `format_proposal_markdown` calls `.get` on it.
- `TypeError` if `proposals` is truthy but not iterable.

Notably, `None` and `[]` are both handled cleanly as “no proposals.” An empty generator is not handled as an empty proposal list because generator objects are truthy.

## Usage Patterns

Typical usage is after proposal generation and optional cascade computation:

```python
proposals = propose_updates(nodes)
cascades = compute_cascades_for_proposals(proposals, nodes)
markdown = format_proposals_file(proposals, nodes, cascades)
```

Caller obligations:

- Ensure every non-empty proposal has `"id"` and the fields required by `format_proposal_markdown`.
- Pass `nodes` if the report should include current belief text.
- Pass `cascades` if the report should include cascade-impact sections; otherwise the report explicitly says cascade impact was not computed.
- Treat the return value as a complete Markdown document string.

## Dependencies

- Internal dependency: `format_proposal_markdown` in the same module.
- No external libraries are used by this function.
- It relies on the proposal dictionary schema produced by `parse_update_proposals` and consumed by `format_proposal_markdown`.

## Topics to Explore

- [function] `reasons/propose_update.py:format_proposal_markdown` — Defines the exact per-proposal Markdown layout, including current text, evidence, cascade impact, and suggested commands.
- [function] `reasons/propose_update.py:propose_updates` — Produces the proposal dictionaries that this formatter consumes, so its batching and validation behavior matters for upstream correctness.
- [file] `tests/test_propose_update.py` — Likely contains edge-case tests for empty proposals, missing cascade data, and rendered Markdown output.
- [general] `cascade-impact-computation` — Explains the expected shape of the `cascades` mapping and how retracted/restored dependents are calculated.
- [general] `proposal-json-schema` — Covers the normalized fields created by `parse_update_proposals`, especially `action`, `failure_mode`, and `basis`.

## Beliefs

- `format-proposals-file-requires-id-key` — For each non-empty proposal, the function directly reads `proposal["id"]`, so a missing `id` raises `KeyError`.
- `format-proposals-file-produces-markdown-document` — The function always returns a newline-joined Markdown string beginning with `# Proposed Updates`.
- `empty-proposals-produces-no-updates-message` — A falsy `proposals` value such as `None` or `[]` returns a document containing `No updates proposed.` with no proposal separators.
- `cascades-lookup-is-keyed-by-proposal-id` — When `cascades` is truthy, cascade impact is looked up via `cascades.get(proposal["id"])`.
- `format-proposals-file-has-no-io-side-effects` — The function only builds and returns a string; it performs no file, network, or mutation side effects.
