---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: strip_frontmatter in reasons/import_beliefs.py

## Purpose

`strip_frontmatter` is a small, dependency-free markdown frontmatter splitter. It detects an optional block at the very start of a markdown string delimited by `---` lines, parses simple `key: value` entries from that block, and returns the remaining body plus a dictionary of string keys to string values.

It exists so `import_into_network` can remove optional frontmatter before parsing beliefs/repos, and can copy selected metadata keys such as `schema_version`, `project_name`, and `created_at` into `network.meta`. It intentionally does **not** use a YAML library and does **not** support real YAML semantics.

## Contract

### Preconditions

- `text` is expected to be a `str`.
- Frontmatter is only recognized if `text` starts with the exact prefix `"---"`.
- The function assumes a closing delimiter is a later line whose `.strip()` equals `"---"`.

### Postconditions

- Always returns a 2-tuple: `(body_text, frontmatter_dict)`.
- If no frontmatter is recognized, returns `(text, {})` unchanged.
- If a closing delimiter is not found, returns `(text, {})` unchanged.
- If frontmatter is recognized:
  - returned body excludes the opening delimiter, all frontmatter lines, and the closing delimiter;
  - returned body is `"\n".join(lines_after_closing_delimiter)`;
  - returned dict contains only parsed `key: value` pairs from frontmatter lines containing `:`.
- The returned dict is always `dict[str, str]`; values are never converted to booleans, numbers, lists, etc.

### Invariants / assumptions

- The opening delimiter is checked only with `text.startswith("---")`, not with exact line equality. A string beginning with `"---foo"` can be treated as frontmatter if a later closing `---` line exists.
- The first subsequent line whose stripped form is `"---"` closes the frontmatter. There is no support for YAML `...`, nested structures, block scalars, lists, comments, or escaping.
- Lines without `:` inside frontmatter are silently ignored.
- Duplicate frontmatter keys are allowed; later assignments overwrite earlier ones.
- An empty key is possible if a line starts with `:` or has only whitespace before the colon.
- An empty dict does not necessarily mean no frontmatter was present: delimiters can exist while every frontmatter line lacks `:`.

## Parameters

### `text: str`

Markdown text that may or may not begin with frontmatter.

Edge cases:

- `""` → no frontmatter, returns `("", {})`.
- `"---"` → no closing delimiter, returns original text and `{}`.
- `"---\n---"` → closing delimiter found; returns `"", {}`.
- `"---\nkey: value\n---\nbody"` → returns `("body", {"key": "value"})`.
- `"---\nkey: value"` → unclosed frontmatter; returns original text and `{}`.
- Leading BOM, blank line, or whitespace before `---` prevents recognition.
- CRLF input is split on `\n`; delimiter detection works because `.strip()` removes `\r`, but body text may retain `\r` characters.

## Return Value

Returns:

```python
tuple[str, dict[str, str]]
```

### Body text

- Original `text` if no frontmatter is recognized.
- Otherwise, the text after the closing `---` line, joined with `\n`.
- May be `""` if the closing delimiter is the final line.
- May start or end with newlines because blank lines after the closing delimiter are preserved.

### Frontmatter dict

- `{}` if no frontmatter is recognized.
- `{}` if frontmatter delimiters exist but no frontmatter line contains `:`.
- Otherwise, a flat dictionary of string keys to string values.
- Keys are stripped of surrounding whitespace.
- Values are stripped of surrounding whitespace, then stripped of leading/trailing double quotes, then leading/trailing single quotes.
- No YAML type conversion or unescaping occurs.

## Algorithm

1. Check whether `text` starts with `"---"`.
   - If not, return `(text, {})`.

2. Split the text into lines with `text.split("\n")`.

3. Search for a closing delimiter starting from line index `1`.
   - A line closes frontmatter if `line.strip() == "---"`.
   - Record the index of the first such line.

4. If no closing delimiter is found, return `(text, {})`.

5. Parse frontmatter lines between the opening line and closing line:
   - For each line containing `":"`:
     - split at the first colon using `partition(":")`;
     - strip whitespace from the key;
     - strip whitespace from the value;
     - strip surrounding double quotes;
     - strip surrounding single quotes;
     - assign to the result dict.

6. Join all lines after the closing delimiter with `"\n"` to form the body.

7. Return `(body, frontmatter_dict)`.

## Side Effects

None. The function is pure with respect to external state:

- does not mutate the input string;
- performs no I/O;
- performs no network calls;
- does not touch global/module state;
- does not mutate any object passed in.

It only allocates local lists, strings, and dictionaries.

## Error Handling

There is no explicit validation or exception handling.

Possible exceptions come from Python itself if the argument type is wrong:

- `text=None` would raise `AttributeError` on `text.startswith(...)`.
- Passing a non-string object may raise `AttributeError` or `TypeError`.

Malformed frontmatter is not treated as an error:

- Unclosed frontmatter is treated as “no frontmatter” and the original text is returned unchanged.
- Frontmatter lines without `:` are ignored.
- Invalid or unusual keys are accepted if the line contains `:`.
- Quoted values are handled only by naive `.strip('"').strip("'")`; no parse error is raised for malformed YAML-like content.

## Usage Patterns

Primary caller in this file:

```python
beliefs_text, frontmatter = strip_frontmatter(beliefs_text)

if frontmatter:
    for key in ("schema_version", "project_name", "created_at"):
        if key in frontmatter and frontmatter[key]:
            network.meta[key] = frontmatter[key]
```

Caller obligations:

- Use the returned body for subsequent markdown parsing (`parse_repos`, `parse_beliefs`).
- Treat returned frontmatter values as raw strings.
- Do not assume `frontmatter` being empty means the file had no delimited frontmatter; it may simply contain no parseable `key: value` lines.
- Do not expect unsupported keys, nested values, lists, booleans, numbers, or escaped strings.

## Dependencies

The function itself depends only on Python built-ins:

- `str.startswith`
- `str.split`
- `str.strip`
- `str.partition`
- `str.join`
- built-in `dict`

It does **not** import or use a YAML parser or any internal module.

## Topics to Explore

- [file] `reasons/import_beliefs.py` — See how frontmatter stripping feeds into the rest of the beliefs import pipeline.
- [function] `reasons/import_beliefs.py:import_into_network` — Caller that copies selected frontmatter keys into `network.meta`.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Parses the post-frontmatter markdown body into claim dictionaries.
- [file] `tests/test_import_beliefs.py` — Tests the expected behavior around frontmatter and import parsing.
- [general] `frontmatter-parsing-edge-cases` — Explore unclosed frontmatter, CRLF handling, duplicate keys, quoted values, and BOM-prefixed files.

## Beliefs

- `strip-frontmatter-requires-leading-dashes` — Frontmatter stripping is only attempted when the input string starts with the exact prefix `"---"`; leading whitespace, blank lines, or a BOM prevent stripping.
- `strip-frontmatter-returns-original-text-when-unclosed` — If no subsequent line strips to `"---"`, the function returns the original text unchanged and an empty dict.
- `strip-frontmatter-produces-body-without-delimiter-lines` — When a closing delimiter is found, the returned body is formed from lines after the closing `---` joined with `\n`, excluding all frontmatter and delimiter lines.
- `strip-frontmatter-parses-flat-colon-pairs` — Frontmatter lines containing `:` are split at the first colon, keys and values are whitespace-stripped, lines without `:` are ignored, and duplicate keys overwrite earlier values.
- `strip-frontmatter-can-produce-empty-dict-with-stripped-body` — If frontmatter delimiters exist but no frontmatter line contains `:`, the function still removes the frontmatter delimiters and returns an empty dictionary.
