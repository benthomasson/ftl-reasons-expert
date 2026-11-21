---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: parse_beliefs in reasons/import_beliefs.py

# `parse_beliefs` — Explanation

## Purpose

`parse_beliefs` is the text-to-structured-data boundary of the beliefs importer. It reads the markdown `beliefs.md` file format (as produced by the companion `beliefs` CLI) and converts it into an in-memory list of claim dictionaries that `import_into_network` can then topologically sort and turn into RMS nodes.

It exists because the on-disk format is human-authored markdown (headers, bullet metadata, a prose body), while the rest of the pipeline wants typed, normalized fields (`status`, `depends_on`, `source_hash`, etc.). This function is the only place in this module that knows the markdown grammar.

## Contract

**Preconditions**
- `text` is a string (typically the *body* of `beliefs.md` after `strip_frontmatter` has removed YAML frontmatter — the caller is expected to do that; `parse_beliefs` itself does not strip frontmatter).
- Nothing else is validated. No claim IDs must be unique; no claim headers are required.

**Postconditions**
- Returns a `list[dict]`. Every dict contains exactly these 14 keys, always present, always initialized:
  `id, status, type, text, source, source_url, source_type, source_hash, date, depends_on, unless, stale_reason, superseded_by, accepted_pr`.
- `depends_on` and `unless` are always `list[str]` (possibly empty).
- All other fields are `str` (possibly empty).
- `status ∈ {"IN", "OUT", "STALE"}` for every returned dict.
- Order is document order: claims appear in the list in the same order their `### ` headers appear in `text`.

**Invariants / implicit assumptions**
- Only *complete* claims are emitted: a new header flushes the previous `current`. If the document ends mid-claim, the trailing `current` is still flushed.
- The "text" field is single-line. Any paragraph continuation after the first prose line is discarded.
- Claim headers must start at **column 0** (`^### `) — no leading whitespace.

## Parameters

| Name | Type | Meaning | Edge cases |
|------|------|---------|------------|
| `text` | `str` | Markdown body of a `beliefs.md` file. | Empty string → returns `[]`. Frontmatter is **not** stripped here — if frontmatter is passed in, lines are simply ignored until the first claim. |

No other parameters. There is no option to relax parsing, no line callback, no error mode.

## Return Value

- `list[dict]` — one dict per claim header. Empty list if no valid headers are present.
- The caller must handle:
  - **Duplicate IDs** — `parse_beliefs` will emit both; dedup happens later in `import_into_network` (`if claim["id"] in network.nodes: skipped += 1`).
  - **Missing dependencies** — `depends_on` may reference IDs that are not in this file; the caller's topological sort treats those as external.
  - **Empty text** — if a claim body was never captured (e.g., it started with a bullet), `text` is `""`.

## Algorithm

Walk through `text.split("\n")` line-by-line with a `current` dict that holds the claim being built.

1. **Header detection (highest priority):**
   ```python
   m = re.match(r"^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)", line)
   ```
   - Group 1 → `id` (any non-whitespace).
   - Group 2 → `status` (must be exactly `IN`, `OUT`, or `STALE`, uppercase).
   - Group 3 → `type` (rest of line, `.strip()`-ed; may be empty).
   - On match: flush `current` into `claims` (if non-`None`), then build a fresh `current` with all defaults.
   - `continue` so the header line is not re-processed by the metadata branches.

2. **Pre-claim guard:** if `current is None`, skip the line. This is why preamble sections (frontmatter that wasn't stripped, `## Repos`, bullets before the first claim) are silently ignored.

3. **Metadata dispatch** — an `if/elif` chain on line prefixes. The order matters:
   - `- Source: `, `- Source URL: `, `- Source hash: ` — simple extraction via `line[len(prefix):].strip()`.
   - `- Date: `
   - `- Depends on: ` — splits on `,`, strips each item, drops empties.
   - `- Unless: ` — same split/strip semantics as `Depends on`.
   - `- Source type: `
   - `- stale reason: ` — matched **case-insensitively** (`line.lower().startswith(...)`), then extracted via `line.index(": ") + 2`. Extracting with `index(": ")` finds the *first* colon+space, which is the separator in the prefix.
   - `- superseded by: ` — same case-insensitive treatment.
   - `- Accepted PR: ` — case-**sensitive** (inconsistent with the two above).

4. **Unknown bullet:** any other `- ` line is dropped (`pass`).

5. **Stray sub-header:** a `### ` line that didn't match the regex is dropped. This is what makes a typo like `### claim-1 [PENDING]` silently disappear rather than become text.

6. **Prose fallback:** if the line is non-empty and `current["text"]` is still `""`, the stripped line becomes the claim's `text`. Only the first such line across the whole claim body is captured.

7. **Final flush:** after the loop, append `current` if non-`None`. Handles the last claim in the file.

## Side Effects

None. Pure function: no I/O, no mutation of `text`, no module-level state, no `network` interaction. The only mutation is the local `claims` list and `current` dict.

## Error Handling

- **No exceptions are raised for malformed input.** Everything degrades silently:
  - Unknown `[STATUS]` → line treated as non-header; a `### ` line falls into the `pass`.
  - Unknown bullet prefix → dropped.
  - Missing `text` → `""`.
  - Bad `Depends on:` format (e.g., trailing comma) → empties filtered out.
- Regex is compiled fresh on every line (`re.match(...)` each iteration) — a minor performance cost, but not a correctness issue.
- Possible `ValueError` from `line.index(": ")` is **impossible in practice** because the `startswith` guard guarantees a `": "` substring exists. Still, it's an implicit coupling between the guard and the extractor.
- `AttributeError` / `TypeError` only if `text` isn't a `str`.

## Usage Patterns

Typical invocation is the middle leg of a three-step pipeline:

```python
body, fm = strip_frontmatter(raw_beliefs_md)
repos = parse_repos(body)
claims = parse_beliefs(body)          # ← this function
import_into_network(network, body, nogoods_text)
```

Caller obligations:
- Strip frontmatter first (`strip_frontmatter`) if the file has any.
- Do **not** rely on `parse_beliefs` for `## Repos` — that's `parse_repos`.
- Handle duplicates and missing deps downstream (as `import_into_network` does).

The function is also called directly by `tests/test_import_beliefs.py`, `tests/test_roundtrip.py`, and `tests/test_source_type.py`, which feed synthetic markdown and assert on the resulting dicts.

## Dependencies

- **Standard library only:** `re` (used inline; no compiled pattern).
- **No intra-package imports** — `parse_beliefs` is leaf-level. This is why it can be tested in isolation and why it lives near the other parsers.

---

## Topics to Explore

- [function] `reasons/import_beliefs.py:import_into_network` — How the parsed claim dicts become RMS nodes and how STALE/OUT become retracted (this is where `unless` and `depends_on` are converted to SL justifications).
- [function] `reasons/import_beliefs.py:parse_nogoods` — The parallel parser for `nogoods.md`; compare how its header regex and metadata handling differ from `parse_beliefs`.
- [function] `reasons/import_beliefs.py:strip_frontmatter` — The step immediately before `parse_beliefs`; explains what this function implicitly ignores in a real `beliefs.md`.
- [file] `tests/test_import_beliefs.py` — Concrete fixtures that pin down header formats, metadata prefixes, and error-tolerance behavior.
- [general] `beliefs-md-format` — The on-disk markdown grammar produced by the beliefs CLI; understanding the writer makes the parser's tolerances and gaps clearer.

## Beliefs

- `parse-beliefs-returns-fixed-key-dicts` — Every dict returned by `parse_beliefs` contains all 14 keys initialized, with `depends_on` and `unless` always being `list[str]` and all other fields `str`.
- `parse-beliefs-silently-skips-malformed-lines` — No input (short of a non-`str` argument) causes `parse_beliefs` to raise; malformed headers, unknown bullets, and stray `### ` lines are dropped silently.
- `parse-beliefs-requires-header-at-column-zero` — A claim header only matches the regex `^### (\S+) \[(IN|OUT|STALE)\]`; any leading whitespace or non-whitelisted status means the line is not treated as a header.
- `parse-beliefs-captures-first-nonbullet-line-as-text` — The claim `text` is set from the first line that is non-empty, doesn't start with `- `, and doesn't start with `### `; all later prose lines are discarded.
- `parse-beliefs-is-a-leaf-dependency` — `parse_beliefs` imports only `re` and no symbols from `reasons.*`, allowing it to be exercised in isolation.
- `parse-beliefs-does-not-strip-frontmatter` — The function assumes its caller already removed YAML frontmatter; if passed a raw file, frontmatter lines are simply skipped via the `current is None` guard.
