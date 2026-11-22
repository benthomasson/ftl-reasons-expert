---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: format_belief_for_update in reasons/propose_update.py

# `format_belief_for_update` (`reasons/propose_update.py`)

## Purpose

Renders a **single TMS belief node into a markdown-ish text block** that gets embedded in the `PROPOSE_UPDATE_PROMPT` sent to an LLM. Each block is meant to give the model enough context to judge whether the belief needs updating/retracting: its text, truth value, source metadata, its full justification structure (antecedents + outlist + label), and the IDs/texts of nodes that depend on it.

It is the per-belief serialization half of the batch flow; `propose_updates` calls it once per belief in a batch, joins the blocks with `"\n\n"`, and interpolates the result into `PROPOSE_UPDATE_PROMPT.format(beliefs=...)`.

## Contract

- **Preconditions (implicit, not type-enforced):**
  - `nodes` is a mapping `id -> node dict`.
  - Each node dict may contain keys `text`, `truth_value`, `source`, `source_url`, `metadata`, `justifications`, `dependents`.
  - Each justification is a dict with `antecedents` (list of ids), `outlist` (list of ids), `label` (str).
- **Postconditions:**
  - Returns a `str`. If `node_id` is missing/`None`-falsy in `nodes`, returns `""`.
  - Otherwise returns a multi-line block beginning with `### {node_id}`, joined by `\n`.
- **Invariants:**
  - Never raises on *well-formed-but-sparse* nodes: every field is read with `.get(..., default)` and downstream lookups use `nodes.get(...)`.
  - No mutation of `nodes` or the node dict.

## Parameters

| Param | Expected type | Edge cases |
|---|---|---|
| `node_id` | hashable (str typically) | If absent from `nodes` or maps to a falsy value (`{}`, `None`), returns `""`. `node_id` itself is not validated; any `;`/`#` chars flow straight into the `###` header. |
| `nodes` | `dict[id, dict]` | Must be a mapping (`.get` is used). Referenced ids that aren't keys produce `"(not found)"` lines. If `nodes` isn't a dict, `nodes.get` raises `AttributeError`. |

## Return Value

- `""` when `node` is falsy.
- Otherwise a `"\n".join(...)` of the assembled `lines` list: header, `Text:`, `Status:` (default `"unknown"`), optional `Source:`, `Source URL:`, `Stale reason:`, then a justification block (or `Type: premise (no justifications)`), then an optional `Dependents (N):` section.
- Callers must handle `""` — `propose_updates` doesn't, so a missing node yields a blank slot in the joined batch text rather than skipping it.

## Algorithm

1. `node = nodes.get(node_id)`; bail out with `""` if falsy.
2. **Header + core fields:** `### {node_id}`, `Text:`, `Status:` (from `truth_value`, default `"unknown"`).
3. **Source fields:** Append `Source:` / `Source URL:` only if truthy (empty strings omitted).
4. **Stale reason:** Read `metadata` defensively (`node.get("metadata", {}) or {}`) so an explicit `None` becomes `{}`; append `Stale reason:` iff `meta["stale_reason"]` is truthy.
5. **Justifications:**
   - If `justs` is truthy, iterate:
     - Emit `Justification {n}/{total}:` **only when `len(justs) > 1`** (single-justification nodes get no header line).
     - If `antecedents`: emit `Antecedents:` then one bullet per id, resolving text via `nodes.get(ant_id)`; unresolved ids become `- {id}: (not found)`.
     - If `outlist`: emit `Unless (must be OUT):` similarly.
     - If `label`: emit `Label: {label}`.
   - Else: emit `Type: premise (no justifications)`.
6. **Dependents:**
   - `dependents = node.get("dependents", [])`.
   - Iterates `sorted(dependents) if isinstance(dependents, (set, list)) else []` — i.e., **only lists and sets are processed**; tuples/other iterables are silently treated as empty.
   - Builds `dep_summaries` of `- {id}: {text}` (or `(not found)`), and emits `Dependents ({len(dep_summaries)}):` + bullets **only if the list is non-empty**. The header count is the number of *resolved summaries*, not necessarily the raw size of `dependents`.
7. Return `"\n".join(lines)`.

## Side Effects

None. Pure function: no I/O, no mutation, no external calls. (The `invoke_model` call happens in `propose_updates`, not here.)

## Error Handling

- Explicit: returns `""` only for a missing node.
- Implicit / unguarded (would raise):
  - `nodes.get(...)` on a non-mapping → `AttributeError`.
  - `node.get(...)` on a non-dict node (e.g., a list) → `AttributeError`.
  - `meta.get(...)` when `metadata` is a truthy non-dict (e.g., a string) → `AttributeError` (the `or {}` only covers `None`/falsy).
  - `j.get(...)` when a justification entry is not a dict → `AttributeError`.
  - `nodes.get(ant_id)` when `ant_id` is unhashable → `TypeError`.
  - Iterating non-iterable `justifications` / non-list-set-iterable `dependents` handles gracefully; only the justification path would raise if `justs` is truthy but non-iterable (effectively impossible for the intended types).
- The function **swallows** missing-node and missing-reference cases into empty string / `(not found)` sentinels rather than raising.

## Usage Patterns

Called inside `propose_updates`:

```python
beliefs_text = "\n\n".join(
    format_belief_for_update(nid, nodes) for nid in batch
)
prompt = PROPOSE_UPDATE_PROMPT.format(beliefs=beliefs_text)
```

Caller obligations:
- Pass the full `nodes` mapping (not just the node) so antecedent/outlist/dependent texts resolve.
- Batch size default is `PROPOSE_UPDATE_BATCH_SIZE = 10`.
- Since output is injected as a string, any `{`/`}` inside belief text is safe here (the prompt template itself uses `{beliefs}`; no second formatting pass on belief bodies).

## Dependencies

- **Internal structural dependency** on the TMS node schema (`text`, `truth_value`, `source`, `source_url`, `metadata.stale_reason`, `justifications[].antecedents/outlist/label`, `dependents`).
- **No module-level imports used inside this function**; the file's imports (`json`, `sys`, `from .llm import invoke_model`) are for sibling functions.

## Assumptions not enforced by the type system

- `dependents` is a `list` or `set`. A `tuple` (or dict) is silently dropped from output — this is a real silent-failure footgun if the TMS ever uses a different container.
- Node dicts are *truthy*; an empty-dict node is treated as missing (returns `""`) and also as `"(not found)"` when referenced from antecedents/outlist/dependents (`if ant_node:` fails on `{}`).
- `metadata` is `None` or a dict; a string metadata value bypasses the `or {}` guard and raises.
- `truth_value` uses the string `"unknown"` as a default; the code doesn't validate against `IN`/`OUT`.
- `justifications` is a list; a single-justification node gets the `Justification` header only if `len > 1`, so downstream consumers can't rely on the header being present just because justifications exist.

## Topics to Explore

- [function] `reasons/propose_update.py:propose_updates` — The caller that batches `IN` beliefs, joins this function's output, and drives the LLM round-trip.
- [function] `reasons/propose_update.py:parse_update_proposals` — Closes the loop: how LLM JSON is sanitized against `FAILURE_MODES`/`BASES`.
- [function] `reasons/llm.py:invoke_model` — The model invocation boundary the batch flow depends on.
- [file] `tests/test_propose_update.py` — Pins expected formatting/edge-case behavior for these functions.
- [general] `tms-justification-semantics` — What antecedents/outlist/label mean and why dependents are included in the prompt for cascade reasoning.

## Beliefs

- `format-belief-for-update-returns-empty-string-for-missing-node` — When `node_id` is not in `nodes` (or maps to a falsy value), the function returns `""`.
- `format-belief-for-update-ignores-non-list-set-dependents` — If `dependents` is not a `list` or `set` (e.g., a tuple or dict), no dependent summaries are produced.
- `format-belief-for-update-marks-justificationless-nodes-as-premises` — Nodes with empty/missing `justifications` emit the line `Type: premise (no justifications)`.
- `justification-numbering-requires-multiple-justifications` — The `Justification N/M:` header is emitted only when `len(justs) > 1`.
- `format-belief-for-update-produces-prompt-fragment` — Its return value is concatenated with `"\n\n"` into the `{beliefs}` slot of `PROPOSE_UPDATE_PROMPT`.
