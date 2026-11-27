---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: format_premise_for_repair in reasons/repair_premises.py

## Purpose

`format_premise_for_repair` is a small formatting helper used to serialize one premise into the text block that will be inserted into `REPAIR_PREMISES_PROMPT`. It combines:

- the premise/node ID,
- the original claim text,
- the review error type and optional review comment,
- an optional source reference,
- the full source document content in a fenced code block.

The function exists so that each premise passed to the LLM repair step has a consistent structure: the model can see the original claim, why it was flagged, and the source material it must use when deciding whether to rewrite or retract the claim.

## Contract

Preconditions, loosely:

- `nodes` is a mapping-like object with `.get()`.
- `node_id` is a key into `nodes`; it may be missing.
- If `nodes.get(node_id)` returns a value, that value is expected to be a mapping-like node object with `.get()`.
- `review_result` is expected to be a mapping-like object with `.get()`. It should not be `None`.
- `source_content` is expected to be a string.

Postconditions:

- If `nodes.get(node_id)` is missing or otherwise falsy, the function returns `""`.
- Otherwise it returns a newline-joined string containing:
  - a `### <node_id>` heading,
  - the original claim,
  - the error type,
  - the review comment only if truthy,
  - the source reference only if truthy,
  - a blank line,
  - `Source document content:`,
  - the source content inside triple-backtick fences.

Invariants:

- The function does not mutate `nodes`, `review_result`, or `source_content`.
- The `Error type:` line is always included when a node exists, even if `review_result` lacks that key.
- `Review comment:` and `Source reference:` are conditional on truthiness.
- `source_content` is appended verbatim and is not escaped or sanitized.

Assumptions not enforced by the type system:

- `nodes`, the node object, and `review_result` all behave like mappings.
- `review_result` is not `None`.
- `source_content` is a string. If it is `None` or another non-string, `"\n".join(lines)` will raise `TypeError`.
- `node_id` can be anything stringifiable for the f-string; normally it is a string.
- The source content does not contain triple backticks that would break the Markdown-style fence.
- The source content is trusted enough not to cause prompt-injection concerns; this function does not sanitize it.

## Parameters

- `node_id`: The premise/node identifier. Used both as the `nodes` lookup key and as the heading text. Expected to be a string in normal use, but any object that works as a dict key and stringifies is accepted by the f-string.
- `nodes`: A mapping of `node_id -> node data`. The node data is expected to be a dict with optional keys:
  - `"text"`: original claim text,
  - `"source"`: source reference/path.
- `source_content`: The full text of the source document associated with the node. It is inserted verbatim inside the fenced block. In the caller, this is either the actual source text or the placeholder `"(source not available)"`.
- `review_result`: A review-result mapping for this premise. Expected keys:
  - `"error_type"`: defaults to `"inaccurate"` if missing,
  - `"comment"`: defaults to `""` if missing.

Edge cases:

- Missing or falsy node: returns `""`.
- Missing `"text"`: uses `""`.
- `"text"` present but `None`: the f-string renders `"None"`, because `.get("text", "")` only defaults when the key is absent.
- Missing `"source"`: omits the source reference line.
- Missing `"comment"`: omits the review comment line.
- Missing `"error_type"`: renders `"inaccurate"`.
- Empty `source_content`: still emits the fenced block, with an empty line inside it.
- Non-string `source_content`: causes `TypeError` when joining lines.

## Return Value

Returns a string.

- If `nodes.get(node_id)` is falsy, returns `""`.
- Otherwise returns the formatted premise block as described above.
- The final string ends with the closing triple-backtick line and does not explicitly add a trailing newline after it, though `source_content` itself may contain trailing newlines.

The caller must handle the possibility of an empty string. In normal `_repair_one` flow, the node is expected to exist, so a non-empty block is expected. However, `format_premise_for_repair` itself silently returns `""` for a missing/falsy node.

## Algorithm

1. Look up `node = nodes.get(node_id)`.
2. If `node` is falsy, return `""` immediately.
3. Start a list of output lines with `### <node_id>`.
4. Append `Original claim: <node.get("text", "")>`.
5. Read `error_type` from `review_result.get("error_type", "inaccurate")`.
6. Read `comment` from `review_result.get("comment", "")`.
7. Append `Error type: <error_type>`.
8. If `comment` is truthy, append `Review comment: <comment>`.
9. Read `source` from `node.get("source", "")`.
10. If `source` is truthy, append `Source reference: <source>`.
11. Append an empty line.
12. Append `Source document content:`.
13. Append an opening code fence: ` ``` `.
14. Append `source_content` directly, without coercion or escaping.
15. Append a closing code fence: ` ``` `.
16. Join all lines with `"\n"` and return the result.

## Side Effects

None.

The function does not mutate its arguments, perform I/O, call an LLM, or modify global state. It only builds and returns a string.

Minor caveat: because it calls `.get()` and uses f-strings, custom mapping objects or objects with unusual `__str__` implementations could technically have side effects, but the function itself performs no intentional side effects.

## Error Handling

There is no `try/except`.

Potential exceptions:

- `AttributeError` if:
  - `nodes` does not have `.get()`,
  - the returned node does not have `.get()`,
  - `review_result` is `None`,
  - `review_result` does not have `.get()`.
- `TypeError` if `source_content` is not a string when `"\n".join(lines)` runs.
- Any exception from stringification via f-strings, if an object’s `__str__` raises.

Missing data is partly tolerated:

- Missing node: returns `""`.
- Missing `text`, `source`, `error_type`, or `comment`: defaults are used.
- Missing `source` reference: line is omitted.
- Missing review comment: line is omitted.

Important caller detail: in `_repair_one`, `format_premise_for_repair(...)` is called before the `try` block that wraps `invoke_model`. Therefore, formatting exceptions are not caught by `_repair_one`’s LLM error handling and can propagate outward.

## Usage Patterns

The normal call site is `_repair_one` in the same module:

```python
prompt_text = format_premise_for_repair(node_id, nodes, source_text, review)
prompt = REPAIR_PREMISES_PROMPT.format(premises=prompt_text)
```

In that flow:

- `review` is usually a dict, defaulting to `{}` when no review result exists.
- `source_text` is the source file content or `"(source not available)"`.
- The returned string is assigned to the `{premises}` placeholder in the repair prompt.

Caller obligations:

- Pass a mapping for `nodes`.
- Ensure the node value is mapping-like when it exists.
- Pass a mapping for `review_result`, not `None`.
- Pass a string for `source_content`.
- Be aware that source content is inserted verbatim into the prompt.
- Consider whether an empty return value should be handled if calling this function independently.

## Dependencies

This function uses only Python built-ins:

- `dict.get`
- f-strings
- `str.join`

It does not import or call external modules. Its broader context depends on `REPAIR_PREMISES_PROMPT`, `_repair_one`, `parse_repair_response`, and `invoke_model` from `reasons/llm.py`, but `format_premise_for_repair` itself is self-contained.

## Topics to Explore

- [function] `reasons/repair_premises.py:_repair_one` — Shows how the formatted premise block is embedded into the LLM prompt and how source text is selected.
- [function] `reasons/repair_premises.py:parse_repair_response` — Explains how the LLM response is parsed and how malformed responses become retractions.
- [file] `tests/test_repair_premises.py` — Reveals expected formatting behavior and edge cases under test.
- [general] `prompt-injection-via-source-content` — The function inserts source text verbatim into an LLM prompt, which is worth reviewing for injection risks.
- [function] `reasons/llm.py:invoke_model` — Documents the model invocation, timeout handling, and exception behavior used by the caller.

## Beliefs

- `format-premise-returns-empty-string-when-node-missing` — `format_premise_for_repair` returns `""` when `nodes.get(node_id)` is missing or otherwise falsy.
- `format-premise-requires-review-result-mapping` — The function calls `review_result.get(...)` without guarding against `None`, so callers must pass a mapping.
- `format-premise-embeds-source-content-verbatim` — `source_content` is appended directly inside a triple-backtick block without escaping backticks or other Markdown.
- `format-premise-has-no-side-effects` — The function only builds and returns a string; it does not mutate inputs or perform I/O.
- `repair-one-formats-before-llm-try-block` — In `_repair_one`, formatting errors occur before the `try` that wraps `invoke_model`, so they are not caught by that error handler.
