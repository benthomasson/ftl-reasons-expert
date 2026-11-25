---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Find where search results are converted into attached evidence, since no direct `chunks.id` to `node_id` link is observed.

# Where Do Search Results Become "Attached Evidence"? (They Don't — Yet)

## Short answer

Based on the observed code, **search results are never converted into attached evidence / network nodes.** There is no `chunks.id → node_id` bridge because the two concepts live in completely separate worlds and are only joined by *text*:

- **Chunks** live in a standalone FTS5 SQLite database (`sources_db`), which may be a totally different `.db` file from the belief network (see the test harness `_create_fts_db` in `tests/test_search_sources.py`).
- **Nodes** live in the `Network.nodes` dict (`reasons/network.py`), keyed by `node_id`.
- The only place a chunk is "consumed" is as plain text formatted into an LLM prompt.

## The actual flow

### 1. The only chunk SQL join is *within the sources DB*
In `reasons/ask.py`, `search_source_chunks` (around line 233–262):

```python
SELECT c.text, c.cluster, c.filename, c.section
FROM chunks_fts
JOIN chunks c ON c.id = chunks_fts.rowid
WHERE chunks_fts MATCH ?
ORDER BY chunks_fts.rank
LIMIT ?
```

The join at **`reasons/ask.py:240`** (`FROM chunks_fts`) and **`reasons/ask.py:241`** (`JOIN chunks c ON c.id = chunks_fts.rowid`) is the *only* join that mentions `chunks`. It links the FTS virtual-table `rowid` to the `chunks.id` primary key — **both inside the same sources database**. It never touches a node table or `node_id`. This is consistent with `find_chunk_node_join_logic` returning exactly those two lines and nothing else.

### 2. Results are reformatted into strings, not nodes
`_search_source_chunks` (same file, immediately after) takes the list of dicts and turns them into a text block:

```python
header = f"[{i}] {row['filename']}" + (f" > {row['section']}" if row["section"] else "")
parts.append(f"### {header}\n\n{row['text']}")
```

That string is then injected via `FTS_RAG_PROMPT` / `MERGE_PROMPT` (both defined at the bottom of `reasons/ask.py`) as "Retrieved Documents" / "Answer B (Source Documents)". No `add_node` call, no `Node` object, no `node_id` assignment occurs on this path.

### 3. The CLI path is also output-only
`cmd_search_sources` (against `reasons/cli.py`) is exercised in `tests/test_search_sources.py::TestSearchSourcesCli` — text and JSON output are asserted, but nothing checks that a node was persisted. Tests `test_returns_list_of_dicts` assert keys `("text", "filename", "section")` — there is no `node_id` or `chunk_id` in the contract.

### 4. Provenance that *is* attached to nodes comes from elsewhere
Where nodes do carry source information, it arrives through `reasons/api.py::add_node`, which accepts `source`, `source_url`, `source_type`, etc. as **caller-supplied strings** and stores them on `Node` via `Network.add_node` (`reasons/network.py`). Nothing in the shown code derives those strings from `search_source_chunks` output — so this is a manual, orthogonal path.

## Why the absence is real, not just undiscovered

- `find_evidence_attachment_terms` (`attach.*evidence|evidence.*attach|attached_evidence|source_evidence|evidence_node`) returned **0 matches** — no "attach" plumbing exists.
- `find_chunk_references` (`chunks|chunk_id|chunk\b`) returned **0 matches** — which is *suspicious*: `chunks` and `chunks_fts` clearly appear in the shown `reasons/ask.py` and test files. This strongly suggests the pattern searches were run against a narrower path/scope (or a stale snapshot) than the shown file contents, so treat the 0-match results as **suggestive, not conclusive**.
- `find_search_result_terms` returned 0 despite `search_source_chunks(` literally existing — same caveat.

## What's missing to be certain

- Only 320 of 566 lines of `reasons/ask.py` were returned — the call sites of `_search_source_chunks` and the merge mode (`MERGE_PROMPT`) lie beyond that window and should be inspected to confirm results are only ever stringified.
- `reasons/llm.py` was not inspected — this is where `invoke_model` could, in principle, receive structured context; worth checking whether any node-creation happens post-synthesis.
- `reasons/pg.py` (the PostgreSQL backend) was not inspected; a separate evidence-table implementation could exist there.

## Topics to Explore

- [file] `reasons/ask.py` — Read lines 320–566 to find the actual call sites of `_search_source_chunks` and the merge/FTS-RAG mode that consumes `FTS_RAG_PROMPT` and `MERGE_PROMPT`.
- [function] `reasons/api.py:add_node` — Confirm that `source`/`source_url`/`source_type` provenance is caller-supplied, and whether any wrapper (e.g., a "import chunks as nodes" helper) exists elsewhere.
- [file] `reasons/cli.py` — Inspect `cmd_search_sources` and any sibling command that might turn chunk results into nodes.
- [general] `chunks-ingest-pipeline` — Find where the `chunks_fts` DB is populated from source documents (the missing upstream half of the RAG flow).
- [file] `reasons/pg.py` — Check whether the PostgreSQL backend defines an explicit evidence/chunk-to-node table that the SQLite path lacks.

## Beliefs

- `search-source-chunks-returns-no-node-id` — `search_source_chunks` returns dicts with exactly the keys `text`, `cluster`, `filename`, `section`; no `node_id` or `chunk_id` field is produced.
- `chunks-db-is-separate-from-network-db` — FTS chunk search opens its own `sources_db` SQLite file, distinct from the belief-network database that stores `node_id` values.
- `fts-join-is-within-sources-db` — The only chunk-related SQL JOIN is `chunks_fts.rowid → chunks.id`, both inside the sources DB; there is no join crossing into node storage.
- `search-results-feed-prompts-not-nodes` — Search results are formatted as text by `_search_source_chunks` and injected into `FTS_RAG_PROMPT`/`MERGE_PROMPT`; they are never materialized as `Node` objects.
- `node-source-provenance-is-caller-supplied` — Node provenance fields (`source`, `source_url`, `source_type`) are provided by the caller of `api.add_node`, not derived from `search_source_chunks` output.
