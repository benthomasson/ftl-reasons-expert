---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: parse_proposals in reasons/derive.py

# `parse_proposals`

## Purpose

`parse_proposals` converts raw LLM markdown output into structured Python dictionaries that can later be validated against the belief network and applied as derived beliefs.

It exists because the LLM output format changed between versions:

- **New format, v0.10+**:
  ```markdown
  ### DERIVE belief-id
  Claim text
  - Antecedents: a, b
  - Unless: c
  - Mode: ANY
  - Label: rationale
  ```

- **Old format, v0.9**:
  ```markdown
  ### DERIVE: `belief-id`
  Claim text
  - **Antecedents**: a, b
  - **Unless**: c
  - **Label**: rationale
  ```

The function tries the new format first, then falls back to the old format only if the new format produced zero proposals.

It is a **syntactic parser only**. It does not validate that antecedents exist, that IDs are unique, or that proposals are semantically valid. Those checks happen later in `validate_proposals`.

---

## Contract

### Preconditions

- `response` is expected to be a `str`.
- The response may contain zero or more proposal blocks in either supported markdown format.
- The proposal blocks are expected to follow the exact field labels and line structure used by the prompts or by `write_proposals_file`.

There is no runtime type check. Passing `None`, `bytes`, or another non-string type will result in a `TypeError` from `re.finditer`.

### Postconditions

- Returns a `list[dict]`, in the order the proposal blocks appeared.
- If at least one **new-format** proposal matches, the function returns those matching new-format proposals immediately and **does not attempt the old-format parser**.
- If no new-format proposal matches, it attempts the old-format parser and returns those matches.
- Malformed proposal blocks are silently skipped.
- The returned list may be empty.

Each new-format proposal dict has:

```python
{
    "kind": "derive" | "gate",
    "id": str,
    "text": str,
    "antecedents": list[str],
    "unless": list[str],
    "mode": "all" | "any",
    "label": str,
}
```

Each old-format proposal dict has the same keys **except `mode` is absent**:

```python
{
    "kind": "derive" | "gate",
    "id": str,
    "text": str,
    "antecedents": list[str],
    "unless": list[str],
    "label": str,
}
```

This difference is important: old-format proposals do not carry a `mode` key. Callers that care about mode must use `p.get("mode", "all")`.

### Invariants

- `kind` is always lowercase: `"derive"` or `"gate"`.
- New-format `mode` is lowercase: `"all"` or `"any"`.
- If a new-format proposal does not include a `Mode:` line, `mode` defaults to `"all"`.
- Antecedents and `unless` values are split on commas and stripped of surrounding whitespace and backticks.
- Empty strings can appear in `antecedents` or `unless` if the LLM emits stray commas, e.g. `"a, b,"`.
- No deduplication is performed.
- No semantic validation is performed.

### Key assumptions not enforced by the type system

- `response` must be a string.
- Proposal fields must be single-line because the regexes do not use `re.DOTALL`.
- New-format fields must use bare `\n` line endings; CRLF headers such as `### DERIVE id\r\n` will likely fail.
- New-format mode values must be uppercase `ALL` or `ANY` if the `Mode:` line is present. A lowercase `- Mode: all` line will cause that proposal block not to match the new regex.
- New-format proposal blocks must not contain blank lines between fields.
- Old-format headers must use the exact old markdown style with backticks around the ID and bold field labels such as `- **Antecedents**:`.
- The old-format kind detection is heuristic: it examines a 60-character slice around the match start and checks whether the uppercase substring `"GATE"` appears in it.
- Mixed old/new responses are not merged. If any new-format proposal matches, old-format proposals in the same response are ignored.

---

## Parameters

### `response`

- Expected type: `str`.
- The raw LLM response or markdown file content.
- May contain prose before, between, or after proposals; `finditer` scans anywhere in the string.
- An empty string returns `[]`.
- `None` or non-string values raise `TypeError`.

---

## Return Value

Returns a list of proposal dictionaries.

Each dictionary represents one parsed proposal and may contain:

- `kind`: `"derive"` or `"gate"`
- `id`: belief ID, with backticks stripped in new-format parsing
- `text`: claim text, stripped
- `antecedents`: list of antecedent IDs
- `unless`: list of outlist-gated IDs, or `[]`
- `mode`: present only for new-format proposals; `"all"` or `"any"`
- `label`: rationale text, stripped

The caller must:

- Handle an empty list as “no proposals parsed”.
- Run `validate_proposals` before applying anything.
- Treat missing `mode` as `"all"` for old-format proposals.
- Be aware that malformed blocks are not reported individually.

---

## Algorithm

1. Initialize `proposals = []`.

2. Compile the new-format regex and scan the response with `finditer`.

   The new regex captures:

   | Group | Meaning |
   |---|---|
   | 1 | `DERIVE` or `GATE` |
   | 2 | belief ID |
   | 3 | claim text |
   | 4 | antecedents |
   | 5 | optional `Unless` list |
   | 6 | optional `ALL` / `ANY` mode |
   | 7 | label |

3. For each new-format match:
   - Lowercase `kind`.
   - Strip backticks from `id`.
   - Strip claim text.
   - Split antecedents on commas, then strip whitespace and backticks.
   - Split `unless` on commas if present; otherwise use `[]`.
   - Default missing mode to `"all"`.
   - Strip label.
   - Append the proposal dictionary.

4. If any new-format proposals were found, return them immediately.

   This means old-format parsing is only a fallback. New and old proposals are never combined.

5. Compile the old-format regex and scan again.

   The old regex captures:

   | Group | Meaning |
   |---|---|
   | 1 | belief ID inside backticks |
   | 2 | claim text |
   | 3 | antecedents |
   | 4 | optional `Unless` list |
   | 5 | label |

6. For each old-format match:
   - Determine `kind` by looking at `response[match.start() - 30 : match.start() + 30]`.
   - If `"GATE"` appears in that slice, set `kind = "gate"`, otherwise `"derive"`.
   - Build the same proposal dictionary, but **omit `mode`**.
   - Append it.

7. Return the old-format proposal list, which may be empty.

---

## Side Effects

- No I/O.
- No mutation of global state.
- No mutation of the input string.
- Creates a new list and new dictionaries.
- Compiles regex patterns on each call rather than using module-level compiled patterns. This is harmless functionally but slightly wasteful if called very frequently.

---

## Error Handling

- Malformed markdown is not reported as an exception. Blocks that do not match the regex are silently skipped.
- If the LLM changes formatting slightly, e.g. uses lowercase mode values, bold labels in the new format, multiline claim text, or blank lines between fields, the parser may return fewer proposals or none at all.
- Passing a non-string response raises `TypeError` from `re.finditer`.
- The function does not catch downstream validation issues. Missing antecedents, duplicate IDs, fewer than two antecedents, and similar retired beliefs are handled by `validate_proposals`.

---

## Usage Patterns

Typical flow:

```python
proposals = parse_proposals(llm_response)
valid, skipped = validate_proposals(proposals, nodes)
results = apply_proposals(valid, db_path="reasons.db")
```

`parse_proposals` is also designed to round-trip files written by `write_proposals_file`. That function writes the new `### DERIVE id` / `### GATE id` format with `- Mode: ALL`, so `parse_proposals` can read those files again, e.g. for a `reasons accept proposed-derivations.md` workflow.

Caller obligations:

- Check for an empty result and decide whether to retry or log.
- Always validate before applying.
- Use `p.get("mode", "all")` rather than `p["mode"]` if old-format proposals may be present.
- Do not assume mixed old/new responses are merged.

---

## Dependencies

Only the Python standard library `re` module is used directly.

Downstream functions in the same file that consume its output include:

- `validate_proposals`
- `apply_proposals`
- `write_proposals_file`

The parser itself does not call those functions.

---

## Topics to Explore

- [function] `reasons/derive.py:validate_proposals` — Shows the network-level checks applied after parsing, including missing nodes, duplicate IDs, and minimum antecedent count.
- [function] `reasons/derive.py:write_proposals_file` — Writes the new markdown format that `parse_proposals` is expected to round-trip.
- [file] `tests/test_derive.py` — Contains parser tests and likely edge cases for both the new and old formats.
- [function] `reasons/derive.py:apply_proposals` — Demonstrates how `mode`, `unless`, and `label` are passed into `api.add_node`.
- [general] `llm-markdown-format-compatibility` — Explore how the v0.9 and v0.10 formats differ and where strict regex parsing may require retries.

---

## Beliefs

- `parse-proposals-prefers-new-format` — If at least one new-format proposal matches, `parse_proposals` returns only new-format proposals and does not run the old-format regex.
- `old-proposals-lack-mode-key` — Proposals parsed from the old v0.9 format omit the `mode` key; callers must default missing mode to `all`.
- `parse-proposals-does-not-validate-network` — `parse_proposals` only extracts syntax and does not check antecedent existence, minimum antecedent count, ID uniqueness, or duplicate proposals.
- `new-format-requires-single-line-fields` — Because the regexes do not use `re.DOTALL`, claim text and field values must be single-line; multiline fields cause the proposal to be skipped.
- `old-kind-detection-uses-context-window` — Old-format `kind` is inferred by searching for `GATE` in a 60-character slice around the match start, not by a regex capture group.
