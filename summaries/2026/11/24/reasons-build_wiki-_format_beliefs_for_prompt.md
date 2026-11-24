---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _format_beliefs_for_prompt in reasons/build_wiki.py

# `_format_beliefs_for_prompt` — Explanation

## Purpose

This function converts a collection of belief-network nodes into a **plain-text structured block** that gets injected into the `{beliefs}` placeholder of `WIKI_PAGE_PROMPT`. It exists so the LLM receives a compact, uniform representation of each belief — ID, truth status, prose text, upstream dependencies, and downstream supporters — without HTML, markdown links, or other rendering concerns that would distract the model.

It is the LLM-facing counterpart to `_format_node` (which produces *markdown* with links/anchors for direct wiki pages). The two functions intentionally diverge: this one is a "raw signal" view for a model, the other is a rendered view for a reader.

## Contract

**Preconditions**
- `node_ids` is an iterable of comparable, unique node IDs (typically strings).
- `node_details` is a mapping `{node_id: detail}` where each `detail` supports `detail["truth_value"]` and `detail["text"]` (direct key access), and optionally `detail.get("justifications")`, `detail.get("supporting_justification")`, `detail.get("dependents")`.

**Postconditions**
- Returns a `str`. If `node_ids` is empty, or none of its members exist in `node_details`, returns `""`.
- Each formatted node appears as a block:
  ```
  ### <nid>
  Status: <truth_value>
  Text: <text>
  Depends on: a, b, c        # only if antecedents non-empty
  Supports: x, y             # only if dependents non-empty
                             # (empty line separator)
  ```
- The overall output ends with a trailing `\n` because each node appends an empty line before `"\n".join`.

**Invariants**
- Node ordering is deterministic via `sorted(node_ids)`.
- Antecedent and dependent lists are sorted and comma-joined.
- No cross-reference links or metadata are emitted (unlike `_format_node`).
- The function is pure: no I/O, no mutation of inputs.

## Parameters

| Name | Type | Notes |
|------|------|-------|
| `node_ids` | iterable of hashable/comparable IDs | Sorted first. Mixed incomparable types (e.g., `int` + `str`) raise `TypeError` from `sorted`. Duplicates are *not* deduplicated — the same node could be emitted twice. |
| `node_details` | `dict[node_id, dict]` | Only *falsy* entries are skipped, so `{}` or `None` both cause `continue`, but a non-empty dict missing required keys raises `KeyError`. |

## Return Value

Returns a single string, ready to be substituted into `WIKI_PAGE_PROMPT.format(topic=..., beliefs=...)`. The empty string is a valid output when there is nothing to format. Callers do **not** need to strip/normalize whitespace — the prompt template tolerates a trailing blank line.

## Algorithm (step-by-step)

1. Initialize `lines = []`.
2. Iterate `nid` in `sorted(node_ids)`:
   a. `detail = node_details.get(nid)` — if falsy, `continue` (silently drop).
   b. Emit header `### {nid}`.
   c. Emit `Status: {detail['truth_value']}` (direct index).
   d. Emit `Text: {detail['text']}` (direct index).
   e. `justifications = detail.get("justifications", [])`.
   f. `si = detail.get("supporting_justification")`.
   g. **If `si` is `None` or out of bounds** (`not (0 <= si < len(justifications))`): union antecedents across **all** justifications.
   h. **Otherwise**: take only `justifications[si]["antecedents"]`.
   i. If the resulting set is non-empty, emit `Depends on: ...` with sorted, comma-joined IDs.
   j. `dependents = detail.get("dependents", [])`; if truthy, emit `Supports: ...` (sorted).
   k. Append `""` (blank line).
3. Return `"\n".join(lines)`.

The key semantic choice: when a `supporting_justification` is valid, only that justification's antecedents are surfaced. This mirrors the TMS notion of the "active" justification — the model sees the *currently reasoned-through* dependency chain, not every possible one. Note that `_format_node` goes further and emits both "active" and "other" antecedent sets; this function deliberately collapses to just the active set to keep the prompt lean.

## Side Effects

None. No files, no stderr, no global state. All mutations are to local `lines` and freshly-constructed `set`s.

## Error Handling

No `try/except`. Errors surface to the caller:

- `KeyError` — if a detail dict lacks `truth_value` or `text`.
- `TypeError` — if `supporting_justification` is a non-`None` non-int (e.g., a string; `0 <= "x"` fails) or a float that lands in-range in the `0 <= si < n` check but then `justifications[si]` fails; also from `sorted` on mixed types.
- `AttributeError` — if `justifications[si]` or any `j` is not a dict/mapping.

Because it's called inside `generate_wiki_page`, which is itself wrapped in a `try/except` inside `build_wiki` (both the parallel and sequential branches), a per-topic failure is logged as `WARN: {label} failed` and the page falls back to `_format_node` rendering — so a malformed node only degrades the LLM synthesis for that topic, it does not abort the whole build.

## Usage Patterns

The only caller is `generate_wiki_page`:

```python
beliefs_text = _format_beliefs_for_prompt(node_ids, node_details)
prompt = WIKI_PAGE_PROMPT.format(topic=topic, beliefs=beliefs_text)
return invoke_model(prompt, model=model, timeout=timeout)
```

Caller obligations:
- `node_ids` should already be the group for one topic page (as produced by `_assign_topics` / `build_wiki`).
- `node_details` should be the same full `{node_id: show_node}` map used across the build so `dependents` and antecedents reference real beliefs.
- Be aware that if the model fails, no annotations are needed — the caller handles exceptions at the `build_wiki` level.

## Dependencies

- **Built-ins only**: `sorted`, `set`, `str.join`, `dict.get`, f-strings.
- **No imports** from the surrounding module (`os`, `re`, `sys`, `reasons.llm`) are used by this function.
- **Schema coupling** to whatever produces `show_node` dicts (the `"truth_value"`, `"text"`, `"justifications"`, `"supporting_justification"`, `"dependents"` keys). This is the biggest hidden contract — nothing in the function's signature enforces it.

**Assumptions not enforced by the type system**
- `supporting_justification`, if not `None`, is an `int` index (not a slice, string, or other sentinel like `-1` meaning "none" — code would silently aggregate all in the `-1` case).
- Each `justification` is a dict with an `antecedents` key that is iterable of node IDs.
- Node IDs are strings that sort deterministically (matching how `_linkify` later uses length-ordered matching).
- The same node dict has already been validated for `truth_value` / `text` upstream (e.g., in `build_wiki`).

## Topics to Explore

- [function] `reasons/build_wiki.py:generate_wiki_page` — The sole caller; wraps this output into the LLM prompt.
- [function] `reasons/build_wiki.py:_format_node` — Sibling markdown formatter; compare how it separates "active" vs "other" antecedents.
- [file] `reasons/build_wiki.py` — For `WIKI_PAGE_PROMPT` guidance and how the fallback path in `build_wiki` uses `_format_node` when this function's caller fails.
- [file] `tests/test_build_wiki.py` — Shows expected output shape and any edge-case fixtures for missing/invalid node dicts.
- [general] `belief-network-node-schema` — The actual shape of the `detail` dicts (`truth_value`, `justifications`, `dependents`, `supporting_justification`) as produced upstream.

## Beliefs

- `_format_beliefs_for_prompt-produces-plain-text-block` — It emits `### <nid>`-headed plain-text blocks with no markdown links or anchors, unlike `_format_node`.
- `supporting-justification-narrows-antecedents` — When `supporting_justification` is a valid index, only that justification's antecedents are emitted; otherwise all justifications are aggregated.
- `function-skips-falsy-node-details` — Any node ID whose `node_details` entry is falsy (`None`, `{}`) is silently dropped from output.
- `truth-value-and-text-are-required-keys` — Direct indexing of `detail['truth_value']` and `detail['text']` means a missing key raises `KeyError` rather than being handled.
- `output-ends-with-trailing-newline` — Each node block ends with an empty line, so `"\n".join(lines)` yields a string ending in `\n`.
- `build-wiki-requires-this-function-to-be-exception-safe-enough-to-fallback` — Because it has no internal error handling, `build_wiki` relies on its own `try/except` around `generate_wiki_page` to fall back to `_format_node` on failure.
