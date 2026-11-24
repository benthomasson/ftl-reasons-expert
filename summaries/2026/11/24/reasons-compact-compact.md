---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: compact in reasons/compact.py

# `compact` — token-budgeted belief-state summary

## Purpose

`compact` produces a Markdown-ish, token-budgeted summary of a `Network` belief state. It exists to make an RMS network small enough to inject into an LLM context window or a file such as `CLAUDE.md`.

It prioritizes information in this order:

1. Nogoods
2. OUT / retracted nodes
3. IN / active nodes, sorted by dependent count

The budget is approximate and soft: it uses a `chars/4` BPE approximation, not a real tokenizer.

## Contract

### Preconditions

The function assumes, but does not validate, that:

- `network` has:
  - `network.nodes`: a mapping whose `.values()` yield node objects
  - `network.nogoods`: a sequence of nogood objects
- Each node has:
  - `truth_value`
  - `text`
  - `id`
  - `dependents`
  - `metadata` with dict-like `.get`
  - `justifications`, where each justification has `.antecedents` and `.label`
- Each nogood has:
  - `id`
  - `nodes`, iterable of strings
  - `resolution`, optional / string-like when truthy
- `budget` is usable in numeric comparison with token estimates.
- `truncate` is treated as truthy/falsy.

### Postconditions

- Returns a single string joined by `"\n"`, with no trailing newline.
- The header is always present:
  - date
  - total node count
  - nogood count
  - IN / OUT counts
- Optional sections are included only if their data exists and their section heading passes the budget check.
- The final `Token count: ...` footer is always appended.
- The original `network` and its nodes/nogoods are not mutated.
- The returned string may exceed `budget`; the budget is not a hard guarantee.

### Invariants

- Nodes are partitioned into IN and non-IN. **Any truth value other than exactly `"IN"` is treated as OUT.**
- IN nodes are sorted by `len(node.dependents)` descending before rendering.
- Summary nodes, identified by truthy `metadata["summarizes"]`, are always visible.
- Regular IN nodes whose IDs appear in a summary's `summarizes` list are hidden from the rendered IN list.
- `_char_count` tracks characters for all lines added through `_add_line`, but not the final footer.
- `_current_tokens()` is `max(1, _char_count // 4)`.
- `_over_budget(line)` reserves an approximate footer using `footer_tokens`.
- Omission lines, blank lines, hidden-summary lines, and the final footer are appended without further budget checks.

### Assumptions not enforced by the type system

- `metadata["summarizes"]` is an iterable of node IDs, normally a list.
- `metadata["retract_reason"]`, `metadata["stale_reason"]`, and `metadata["superseded_by"]` are string-like when truthy.
- `nogood.nodes` contains strings.
- `network.nodes` is ordered consistently if callers care about OUT-node order; OUT nodes are not sorted.
- `budget` is an integer-ish value; a negative or very small budget causes most sections to be omitted but does not raise.

## Parameters

### `network: Network`

The RMS network to summarize. The function reads `network.nodes`, `network.nogoods`, and node metadata. It does not modify them.

### `budget: int = 500`

Approximate maximum token budget. Tokens are estimated as `max(1, len(text) // 4)`. Small or negative values cause section headings and lines to fail the budget check. The function still returns the header and footer even if they exceed the budget.

### `truncate: bool = True`

If truthy, node text longer than 80 characters is truncated as:

```python
t[:77] + "..."
```

So a node text of length 81 becomes length 80. Exactly 80 characters is not truncated.

This only affects `_text(node)` usage in IN and OUT node lines. It does not truncate nogood resolutions, superseded-by values, or the 60-character stale/retract reason.

## Return Value

Returns a string containing:

- Header lines
- Optional `## Nogoods` section
- Optional `## OUT (retracted)` section
- Optional `## IN (active)` section
- Final line like:

```text
Token count: ~123 / 500 budget
```

Important nuance: the displayed token count is computed from `_current_tokens()` **before** appending the footer, so it excludes the footer's own characters. The footer reserve is used during budget checks, but the printed count is not the total token estimate of the returned string.

Callers should treat the result as an approximate summary, not a hard-budgeted artifact.

## Algorithm

1. Partition `network.nodes.values()`:
   - `truth_value == "IN"` goes into `in_nodes`
   - everything else goes into `out_nodes`

2. Sort `in_nodes` by `len(node.dependents)` descending.

3. Build header lines:
   - current local date via `date.today().isoformat()`
   - total nodes
   - nogood count
   - IN count
   - OUT count

4. Initialize character accounting:
   - `_char_count` starts as the joined length of the header lines, including newlines.
   - `_add_line` appends a line and adds `1 + len(line)` to `_char_count`.
   - `_current_tokens()` returns `max(1, _char_count // 4)`.
   - `_over_budget(line)` checks:
     ```python
     _current_tokens() + estimate_tokens(line) + footer_tokens > budget
     ```
     where `footer_tokens` is an estimate of a footer string using `budget` twice.

5. Render nogoods if any exist and the heading fits:
   - Add `## Nogoods`.
   - For each nogood, render:
     ```text
     - {id}: {', '.join(nodes)}{ — resolution if present}
     ```
   - If a line does not fit, add an omission line like:
     ```text
       ... (N more nogoods omitted)
     ```
     and break.
   - Add a blank line after the section.

6. Render OUT nodes if any exist and the heading fits:
   - Add `## OUT (retracted)`.
   - For each OUT node, build an optional reason:
     - prefer `metadata["retract_reason"]` or `metadata["stale_reason"]`
     - render as ` (stale: {reason[:60]})`
     - else if `metadata["superseded_by"]` exists, render ` (superseded by: ...)`
   - Render:
     ```text
     - {id}: {truncated_text}{reason}
     ```
   - If over budget, add an omission line and break.
   - Add a blank line after the section.

7. Render IN nodes if any exist and the heading fits:
   - Partition IN nodes into:
     - summary nodes: truthy `metadata["summarizes"]`
     - regular nodes: everything else
   - Collect all IDs covered by summaries into `covered_by_summary`.
   - Build `visible_nodes` as:
     - all summary nodes
     - regular nodes whose IDs are not covered by any summary
   - Sort `visible_nodes` by dependent count descending.
   - `hidden_count = len(in_nodes) - len(visible_nodes)`, effectively counting covered regular nodes.
   - Add `## IN (active)`.
   - For each visible node:
     - prefix `[summary] ` if it is a summary node
     - find the first justification with antecedents and label not `"summarizes"` to render ` <- antecedents`
     - add dependent count if nonzero
     - add `(covers N nodes)` if the node has `summarizes`
     - render:
       ```text
       - {prefix}{id}: {truncated_text}{deps}{dep_info}{sum_info}
       ```
     - if over budget, add an IN omission line and break
   - If `hidden_count` is nonzero, add:
     ```text
       ({hidden_count} nodes hidden by summaries)
     ```
   - Add a blank line after the section.

8. Append the final footer:
   ```text
   Token count: ~{_current_tokens()} / {budget} budget
   ```
   The footer is appended directly, without updating `_char_count`.

9. Return `"\n".join(lines)`.

## Side Effects

- No mutation of `network`, nodes, nogoods, or metadata.
- Sorts local lists only.
- Calls `date.today()`, so output depends on the current local date.
- No file I/O, network I/O, logging, or printing.
- Complexity is roughly `O(n log n)` because IN nodes are sorted.

## Error Handling

There is no explicit error handling and no input validation.

Exceptions propagate naturally, including:

- `AttributeError` if expected node / nogood / metadata fields are missing.
- `TypeError` if `budget` is not comparable to numbers.
- `TypeError` if truthy `retract_reason` is not sliceable.
- `TypeError` if truthy `summarizes` is not iterable.
- `AttributeError` if `metadata` is `None` or lacks `.get`.
- `TypeError` if `nogood.nodes` is not iterable.

The function does **not** raise when the result exceeds `budget`. It silently returns an over-budget string. This can happen because:

- omission lines are added without budget checks
- blank lines are added without budget checks
- the hidden-summary line is added without budget checks
- the final footer is always appended
- `_over_budget` ignores newline characters when estimating the added line

If a section heading does not fit, the entire section is omitted with no omission marker.

## Usage Patterns

Typical usage:

```python
summary = compact(network)
summary = compact(network, budget=200)
summary = compact(network, budget=1000, truncate=False)
```

Callers usually embed the returned string into an LLM prompt or documentation file. Because the budget is soft, callers that need a hard limit should measure the returned string themselves or use a real tokenizer.

The caller is responsible for providing a `Network` whose node and nogood shapes match the assumptions above.

## Dependencies

- `datetime.date` for the current date.
- `reasons.network.Network` for the input type.
- Local `estimate_tokens` from `reasons/compact.py`.
- Runtime node/nogood fields:
  - `node.truth_value`, `node.text`, `node.id`, `node.dependents`, `node.metadata`, `node.justifications`
  - `nogood.id`, `nogood.nodes`, `nogood.resolution`

## Topics to Explore

- [file] `reasons/network.py` — Defines `Network`, node fields, metadata conventions, and nogood structures that `compact` assumes.
- [function] `reasons/compact.py:estimate_tokens` — The `chars/4` heuristic behind every budget check and printed token count.
- [file] `tests/test_compact.py` — Likely captures expected formatting, truncation, omission behavior, and soft-budget edge cases.
- [general] `soft-token-budget-behavior` — How section omission, omission lines, footer reserve, and final footer interact to allow over-budget output.
- [general] `summary-node-coverage` — How `metadata["summarizes"]` hides regular IN nodes while always keeping summary nodes visible.

## Beliefs

- `non-in-nodes-are-treated-as-out` — Every node whose `truth_value` is not exactly `"IN"` is classified into `out_nodes`, not only nodes with `truth_value == "OUT"`.
- `budget-is-soft` — The returned string can exceed `budget` because omission lines, blank lines, hidden-summary lines, and the final footer are appended without budget checks.
- `footer-token-count-excludes-footer` — The final `Token count` line reports `_current_tokens()` before appending the footer, so it excludes the footer's own characters.
- `compact-does-not-mutate-network` — `compact` reads and sorts local lists but performs no writes to `network`, node metadata, or nogoods.
- `in-nodes-sorted-by-dependent-count` — Visible IN nodes are sorted descending by `len(node.dependents)`, while summary nodes are always included and covered regular nodes are hidden.
