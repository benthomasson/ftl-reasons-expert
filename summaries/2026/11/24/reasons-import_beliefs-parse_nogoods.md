---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: parse_nogoods in reasons/import_beliefs.py

## `parse_nogoods`

### Purpose

`parse_nogoods` is a pure markdown parser for `nogoods.md` content. It converts a text file into a list of plain dictionaries, one per matching nogood header.

It exists as a helper for `import_into_network`: the importer calls it, validates/filters the affected node IDs against the actual network, and then constructs `Nogood` objects. The parser itself does not know about `Network`, `Nogood`, or node validity.

### Contract

Preconditions:

- `text` is a `str`.
- Matching nogood sections use this header shape:
  ```markdown
  ### nogood-<non-whitespace-id>: <label>
  ```
- Recognized metadata lines use exact prefixes:
  ```markdown
  - Discovered: ...
  - Resolution: ...
  - Affects: a, b, c
  ```

Postconditions:

- Returns `list[dict]`.
- Each matched header produces one dictionary with these keys:
  - `id`: captured header ID, including the `nogood-` prefix
  - `label`: text after the header colon, stripped
  - `discovered`: string, default `""`
  - `resolution`: string, default `""`
  - `affects`: list of strings, default `[]`
- Ordering follows the order of matching headers in the input.
- Malformed or unrecognized lines are ignored.
- No validation is performed on IDs, affected nodes, duplicates, or required field presence.

Invariants:

- `current` holds the dictionary for the most recently matched header.
- When a new matching header is found, the previous `current` is appended if present.
- At EOF, any remaining `current` is appended.
- Later matching metadata lines overwrite earlier values for the same field.
- Multiple `Affects:` lines replace the previous list rather than merging.

### Parameters

`text: str`

The full text content of a nogoods markdown file. This function does not accept a file path and performs no file I/O.

Edge cases:

- Empty string returns `[]`.
- Text with no matching `### nogood-...:` headers returns `[]`.
- A header with no label is accepted; `label` becomes `""`.
- A nogood with no metadata lines returns defaults: empty strings and empty list.
- Headers that do not start with `nogood-` are ignored.
- Lines before the first matching header are ignored because `current is None`.
- Non-string input would raise before meaningful parsing, e.g. `None.split(...)` raises `AttributeError`.

### Return Value

Returns `list[dict]`. Each dictionary has exactly the five keys listed above, because dictionaries are created with those keys and only those fields are mutated.

The caller must not assume:

- that `affects` contains valid network node IDs,
- that `affects` has at least two entries,
- that IDs are unique,
- that all sections in the source file were parsed,
- that arbitrary `###` headings terminate a nogood section.

The full-file caller, `import_into_network`, later filters `affects` against `network.nodes` and only creates a `Nogood` if at least two valid affected nodes remain.

### Algorithm

1. Initialize `nogoods = []` and `current = None`.
2. Split the input text on `"\n"`.
3. For each line:
   - Try to match:
     ```python
     r"^### (nogood-\S+?):\s*(.*)"
     ```
   - If it matches:
     - Append the previous `current` if one exists.
     - Create a new dictionary from `m.group(1)` and `m.group(2).strip()`.
     - Initialize `discovered`, `resolution`, and `affects`.
     - Continue to the next line.
   - If no current nogood exists, skip the line.
   - If the line starts with `- Discovered: `, store the stripped remainder.
   - If the line starts with `- Resolution: `, store the stripped remainder.
   - If the line starts with `- Affects: `, strip the remainder, split on commas, strip each item, and keep non-empty items.
   - Otherwise ignore the line.
4. After the loop, append any remaining `current`.
5. Return the list.

The regex captures the ID lazily up to the first colon that can satisfy the pattern. In practice, IDs containing colons will be split at the first colon, so IDs should not contain `:` or whitespace.

### Side Effects

None beyond allocating new lists and dictionaries.

The function:

- does not mutate `text`,
- does not read files,
- does not write files,
- does not modify a `Network`,
- does not emit logs or warnings.

### Error Handling

There is no explicit error handling. Malformed content is silently skipped.

Potential exceptions:

- Passing a non-string to `text` will raise a standard Python exception when `.split()` is called.
- The regex itself is constant and valid, so `re.match` will not raise for normal string input.

No validation errors are returned. For example, an `Affects:` line with invalid node IDs is still parsed into the `affects` list; validation happens later in `import_into_network`.

### Usage Patterns

Typical call:

```python
nogoods = parse_nogoods(nogoods_text)
```

In this module, it is called from `import_into_network` only when `nogoods_text` is truthy:

```python
nogoods = parse_nogoods(nogoods_text)
```

The caller then:

- skips nogoods whose IDs already exist in `network.nogoods`,
- filters `ng["affects"]` to only nodes present in `network.nodes`,
- imports only nogoods with at least two valid affected nodes.

Caller obligations:

- Provide markdown text, not a path.
- Handle an empty result.
- Perform validation if strict nogood semantics are required.
- Be aware that non-nogood `###` sections do not reset `current`; metadata-like lines under later headings can still mutate the prior nogood.

### Dependencies

- Standard library: `re`
- No internal imports are used by `parse_nogoods` itself.
- In the full module, the parsed dictionaries are consumed by `import_into_network`, which depends on `Nogood` and `Network`.

### Assumptions Not Enforced by Types

- Matching headers always start with `### nogood-`.
- Nogood IDs contain no whitespace and no colon.
- Metadata lines use exact, case-sensitive prefixes including the trailing space after the colon.
- `Affects` entries are comma-separated node IDs, and node IDs do not contain commas.
- A non-nogood `###` heading should not appear between a nogood header and its metadata, because the parser does not close `current` on other headings.
- Duplicate IDs are preserved by this function; duplicate filtering is the caller’s responsibility.
- Later duplicate metadata lines overwrite earlier ones.

## Topics to Explore

- [function] `reasons/import_beliefs.py:import_into_network` — Shows how parsed nogood dictionaries are validated, filtered against network nodes, and converted into `Nogood` objects.
- [file] `tests/test_import_beliefs.py` — Exercises parsing/import behavior and likely documents expected edge cases.
- [file] `tests/test_roundtrip.py` — Verifies whether imported beliefs/nogoods survive serialization and re-import.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Similar line-oriented markdown parser for claims; useful for comparing parsing conventions and assumptions.
- [general] `nogoods-markdown-schema` — The parser hardcodes header and field formats, so the authoritative schema and section-boundary rules are worth verifying.

## Beliefs

- `parse-nogoods-requires-nogood-prefix` — Only lines matching `### nogood-<non-whitespace>: ...` start a parsed nogood section.
- `parse-nogoods-parses-exact-field-prefixes` — Only lines beginning with `- Discovered: `, `- Resolution: `, or `- Affects: ` are recognized as metadata.
- `parse-nogoods-splits-affects-on-commas` — The `Affects` value is comma-split, stripped per item, and empty items are removed.
- `parse-nogoods-does-not-validate-affects` — It does not check node existence, deduplicate IDs, or require at least two affected nodes; `import_into_network` performs those checks.
- `parse-nogoods-preserves-header-order` — Returned nogood dictionaries appear in the same order as their matching headers in the input text.
