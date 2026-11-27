---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search for and test import handling of `- Duplicate of:`; no importer occurrence appears in the provided `duplicate_of` search results.

## Summary

The codebase currently **exports** `- Duplicate of:` as a human-readable markdown line but does **not import** it. The `duplicate_of` metadata search found the setter/consumer/exporter/tests, but no occurrence in either importer implementation:

- `reasons/import_agent.py` does not reference `duplicate_of`.
- `reasons/import_beliefs.py` does not reference `duplicate_of`.
- No test exercises importing a `- Duplicate of:` markdown line.

There is one search inconsistency worth noting: `locate_duplicate_of_marker` returned zero matches, yet `locate_duplicate_of_metadata` shows `reasons/export_markdown.py:106-107` writing exactly `- Duplicate of: {node.metadata['duplicate_of']}`. The marker search was likely scoped/escaped differently, but the metadata search and file reads are enough to confirm the importer gap.

## Evidence From the Code

### The marker is produced on export

`reasons/export_markdown.py:106-107`:

```python
if node.metadata.get("duplicate_of"):
    lines.append(f"- Duplicate of: {node.metadata['duplicate_of']}")
```

So markdown export emits:

```markdown
- Duplicate of: <canonical-id>
```

when a node has `metadata["duplicate_of"]`.

### The metadata is set and consumed elsewhere

- `reasons/api.py:1267` sets `meta["duplicate_of"] = canonical_id`.
- `reasons/build_wiki.py:62-63` reads `metadata["duplicate_of"]` to resolve a canonical ID.
- `reasons/cli.py:170` prints `duplicate_of` metadata when a node is retracted/deduplicated.
- `tests/test_deduplication.py:26,147,165-166,210` asserts `duplicate_of` metadata after deduplication.
- `tests/test_build_wiki.py:129-147` tests wiki behavior for `duplicate_of`.

These confirm the metadata key exists and is used outside import.

### The markdown importer ignores the marker

`reasons/import_beliefs.py:parse_beliefs` parses specific metadata lines such as `- Source:`, `- Date:`, `- Depends on:`, `- Unless:`, `- Superseded by:`, etc. For unrecognized metadata lines, it falls through to:

```python
elif line.startswith("- "):
    pass  # other metadata lines — skip
```

That means:

```markdown
- Duplicate of: canonical-node
```

is silently skipped by `parse_beliefs`. There is no branch producing a `duplicate_of` key.

Even if `parse_beliefs` did capture it, `reasons/import_beliefs.py:import_into_network` builds metadata only from selected fields:

```python
metadata = {}
if claim["type"]:
    metadata["beliefs_type"] = claim["type"]
if claim.get("source_type"):
    metadata["source_type"] = claim["source_type"]
if claim["stale_reason"]:
    metadata["stale_reason"] = claim["stale_reason"]
if claim["superseded_by"]:
    metadata["superseded_by"] = claim["superseded_by"]
if claim.get("accepted_pr"):
    metadata["accepted_pr"] = claim["accepted_pr"]
```

It does not include `duplicate_of`.

### The agent markdown importer also drops it

`reasons/import_agent.py:_normalize_markdown` calls `parse_beliefs(beliefs_text)` but then constructs metadata as:

```python
meta = {}
if c["type"]:
    meta["beliefs_type"] = c["type"]
```

So `_normalize_markdown` also has no path for `duplicate_of`, even if the parser were updated.

### JSON import preserves it

`reasons/import_agent.py:_normalize_json` does:

```python
meta = dict(ndata.get("metadata", {}))
```

Because this copies the entire metadata dict from JSON, `duplicate_of` would survive a JSON import if it was present in the JSON `metadata` field. The loss is specific to markdown import.

## Test Coverage Gap

The existing "duplicate" import tests are about **skipping already-existing nodes**, not about `- Duplicate of:`:

- `tests/test_import_agent.py:test_import_agent_skip_duplicates` re-imports the same beliefs and asserts `claims_skipped == 3`.
- `tests/test_import_beliefs.py:TestImportIntoNetwork.test_skip_duplicates` re-imports the same beliefs and asserts all claims are skipped.
- `tests/test_import_json.py:TestImportJson.test_skip_duplicates` imports the same JSON twice and asserts `nodes_imported == 0`.

None of these tests create a node with `metadata["duplicate_of"]`, export markdown, re-import it, and assert the metadata survives. A targeted test should:

1. Create/import a node with `metadata["duplicate_of"] = "canonical-id"`.
2. Export markdown via `reasons/export_markdown.py`.
3. Import that markdown via `import_beliefs.import_into_network` or `import_agent`.
4. Assert the resulting node metadata contains `duplicate_of == "canonical-id"`.

Given current code, that test should fail for markdown import.

## Likely Fix Points

To support markdown round-tripping of `- Duplicate of:`:

- Add a branch in `reasons/import_beliefs.py:parse_beliefs` for `- Duplicate of: ` and store it on the claim dict.
- Add `duplicate_of` to the metadata construction in `reasons/import_beliefs.py:import_into_network`.
- Add `duplicate_of` propagation in `reasons/import_agent.py:_normalize_markdown`.
- Add a round-trip markdown import test.

## Topics to Explore

- [function] `reasons/import_beliefs.py:parse_beliefs` — Where markdown claim metadata is parsed and where `- Duplicate of:` is currently swallowed by the generic `- ` skip branch.
- [function] `reasons/import_beliefs.py:import_into_network` — Builds the final metadata dict for non-agent markdown imports and omits `duplicate_of`.
- [function] `reasons/import_agent.py:_normalize_markdown` — Agent-specific markdown normalization that also omits `duplicate_of` metadata.
- [file] `reasons/export_markdown.py` — Emits `- Duplicate of:` at lines 106-107; useful for building a failing round-trip test.
- [general] `duplicate-of-roundtrip` — A proposed integration test that exports markdown with `duplicate_of` metadata and re-imports it.

## Beliefs

- `export-markdown-emits-duplicate-of-marker` — `reasons/export_markdown.py:106-107` appends `- Duplicate of: <id>` when `node.metadata["duplicate_of"]` is set.
- `markdown-parser-ignores-duplicate-of-marker` — `reasons/import_beliefs.py:parse_beliefs` has no branch for `- Duplicate of:`; the generic `elif line.startswith("- "): pass` skips it.
- `import-beliefs-drops-duplicate-of-metadata` — `reasons/import_beliefs.py:import_into_network` builds metadata from selected fields and does not include `duplicate_of`.
- `import-agent-markdown-drops-duplicate-of-metadata` — `reasons/import_agent.py:_normalize_markdown` only copies `beliefs_type` into metadata, so it cannot preserve `duplicate_of` from markdown.
- `json-import-preserves-duplicate-of-metadata` — `reasons/import_agent.py:_normalize_json` copies `ndata.get("metadata", {})`, so `duplicate_of` survives JSON import.
