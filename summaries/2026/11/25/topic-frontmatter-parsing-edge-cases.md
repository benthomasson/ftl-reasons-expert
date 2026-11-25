---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore unclosed frontmatter, CRLF handling, duplicate keys, quoted values, and BOM-prefixed files.

## What the code shows

The frontmatter path is centered on `reasons/import_beliefs.py:strip_frontmatter` (`reasons/import_beliefs.py:21`). Its docstring explicitly describes the frontmatter as a “simple” format and says it returns `(body_text, frontmatter_dict)`, with `(text, {})` when no frontmatter is present (`reasons/import_beliefs.py:22-26`). It is used by both import paths:

- `reasons/import_beliefs.py:180` — strips frontmatter before parsing beliefs and copies keys into `network.meta`.
- `reasons/pg.py:2395` — strips frontmatter before import and writes selected keys into domain metadata.

Export writes frontmatter-like YAML in `reasons/export_markdown.py:15` and `reasons/export_card.py:65`. The metadata fields come from `reasons/metadata.py:build_meta`, which emits `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, and `generator`. Tests expect some values quoted in markdown, e.g. `'schema_version: "1.0"'` and `'project_name: "test-proj"'` in `tests/test_metadata.py`.

The important gap: the observations do **not** include the body of `strip_frontmatter`, so the exact parser behavior for malformed input is not directly visible. The visible tests confirm only a few cases, especially ordinary frontmatter and double-quoted values.

## Edge cases

### Unclosed frontmatter

No visible test covers an unclosed `---` block. The documented contract only says “If no frontmatter is present, returns `(text, {})`” (`reasons/import_beliefs.py:26`). It does **not** state what happens when a file starts with `---` but never has a closing delimiter.

Because the parser body is not shown, we cannot safely infer whether an unclosed block is treated as:

- no frontmatter and left in the body,
- frontmatter parsed through end-of-file, or
- some partial/failed parse.

This must be checked in `reasons/import_beliefs.py:strip_frontmatter`.

### CRLF handling

There are no explicit CRLF/newline-normalization matches in the observations other than an unrelated comment in `reasons/compact.py:66`. The frontmatter tests use `\n` delimiters, e.g.:

```python
'---\nschema_version: "1.0"\nproject_name: test\n---\n\n# Beliefs'
```

If `strip_frontmatter` splits only on `"\n"` and compares exact lines against `"---"`, then CRLF input would produce lines ending in `\r`, and the closing delimiter check could fail. If it uses `splitlines()` or strips `\r`, CRLF may work. The observations are insufficient to determine which.

### Duplicate keys

No frontmatter-specific duplicate-key handling appears in the observations. The duplicate-key search hits are about duplicate **nodes** in `reasons/api.py:mark_duplicate` and `reasons/api.py:deduplicate`, not frontmatter keys.

Also, there is no evidence of `yaml.safe_load` or a real YAML parser being used for frontmatter. The observed frontmatter search only finds `strip_frontmatter`, and the docstring calls it “simple.” That suggests duplicate frontmatter keys are likely not detected or rejected explicitly. If implemented with a plain `dict[key] = value`, the last occurrence wins. That behavior is plausible but not proven from the provided snippets.

### Quoted values

Double-quoted values are explicitly covered. In `tests/test_metadata.py`, `TestStripFrontmatter.test_quoted_values` uses:

```python
text = '---\nupdated_at: "2026-05-23T14:00:00Z"\n---\nContent'
body, fm = strip_frontmatter(text)
assert fm["updated_at"] == "2026-05-23T14:00:00Z"
```

So the parser is expected to remove surrounding double quotes from values. Markdown export also emits quoted metadata such as `schema_version: "1.0"` and `project_name: "test-proj"`, so the importer must unquote those or round-trips would produce mismatched metadata.

`tests/test_metadata_extended.py` has a module docstring claiming coverage of “Frontmatter edge cases (partial, malformed, single-quoted)”, but only the first 200 lines were returned, so the actual single-quoted/malformed test bodies are not visible. We can say double-quote stripping is confirmed; single-quote behavior needs inspection.

### BOM-prefixed files

No `utf-8-sig`, `BOM`, or `bom` matches were found. There is no observed code that strips a Unicode BOM before frontmatter detection.

If a file starts with a BOM and is read as plain UTF-8, the first character may be `\ufeff`, so a check like `text.startswith("---")` would fail. That would cause the frontmatter to be treated as normal body text unless the reader uses `encoding="utf-8-sig"` or explicitly strips the BOM. The observations do not show whether that is handled, so this is an unverified risk.

## Summary

The code advertises YAML frontmatter, but the observed implementation is a simple key/value parser using `strip_frontmatter`. Quoted double values are tested. Unclosed blocks, CRLF normalization, duplicate key semantics, single-quoted values, and BOM handling are not conclusively resolved by the provided snippets. The missing piece is the full implementation of `reasons/import_beliefs.py:strip_frontmatter` and the remainder of `tests/test_metadata_extended.py`.

## Topics to Explore

- [function] `reasons/import_beliefs.py:strip_frontmatter` — Inspect the actual delimiter detection, line splitting, quote stripping, duplicate-key handling, and treatment of malformed/unclosed blocks.
- [file] `tests/test_metadata_extended.py` — Read the rest of the file for the promised partial, malformed, and single-quoted frontmatter tests.
- [file] `reasons/export_markdown.py` — Confirm exact frontmatter serialization, quoting rules, and newline output.
- [file] `reasons/import_beliefs.py` — Check how files are read and whether BOM/encoding handling exists outside `strip_frontmatter`.
- [general] `frontmatter-edge-case-coverage` — Determine whether unclosed frontmatter, CRLF, BOM, and duplicate keys have tests or only undocumented behavior.

## Beliefs

- `frontmatter-is-parsed-by-simple-parser` — `strip_frontmatter` is documented as a simple frontmatter format parser, not a full YAML parser.
- `frontmatter-returns-body-and-dict` — `strip_frontmatter` returns `(body_text, dict[str, str])` and returns `(text, {})` when no frontmatter is present.
- `frontmatter-metadata-propagates-to-network-meta` — Import code copies selected frontmatter keys into `network.meta` via `frontmatter[key]` at `reasons/import_beliefs.py:180-185`.
- `double-quoted-frontmatter-values-are-unquoted` — `TestStripFrontmatter.test_quoted_values` expects `updated_at: "2026-05-23T14:00:00Z"` to become an unquoted dict value.
- `no-explicit-bom-or-crlf-handling-located` — The searched frontmatter-related matches contain no `utf-8-sig`, BOM, or CRLF handling code.
