---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/build_wiki.py

# `reasons/build_wiki.py`

## Purpose

This module is the **static-site generator** for the belief network. It takes the in-memory representation of a Truth Maintenance System (TMS) — a mapping of node IDs to their details (truth value, text, justifications, dependents) — and renders it as a set of interlinked Markdown files organized by topic. Think of it as a `make html` step for your knowledge base: it produces a self-contained directory of `.md` files that you can push to a static host, render in VS Code, or feed to a doc site.

Its single public responsibility is `build_wiki()`: given node details and a grouping, write `index.md` + one page per group to a target directory, optionally enriching each page with LLM-synthesized prose.

## Key Components

### Constants

| Name | Role |
|---|---|
| `_TOPIC_STOP_WORDS` | A frozenset of ~60 English function words. Used (implicitly by callers) when extracting meaningful tokens from node IDs for topic matching. Not directly referenced inside this file's functions, but it signals the vocabulary the caller is expected to filter before calling `_assign_topics`. |
| `WIKI_PAGE_PROMPT` | A multi-line f-string template with two placeholders (`{topic}`, `{beliefs}`). Instructs the LLM to write an encyclopedic narrative, not a list. |
| `_RESERVED_SLUGS` | `{"index"}` — prevents a topic literally named "index" from colliding with the mandatory `index.md`. |

### Functions (internal)

- **`_assign_topics(node_ids, topics)`** → `dict[str, list[str]]`
  Splits each node ID on `[-._:]`, takes tokens longer than 2 chars, and assigns the node to the **first** topic whose label appears as a token. Unmatched nodes land in `"Other"`. Returns only non-empty groups. This is a *caller-side* utility — `build_wiki` receives pre-grouped data.

- **`_page_name(label)`** → `str`
  Lowercases, replaces non-alphanumerics with `-`, strips edge dashes. Falls back to `"other"` if the result is empty. This is the slug that becomes the filename.

- **`_format_node(node_id, node_detail, node_to_page, all_details=None)`** → `str`
  The workhorse renderer for a single belief. It emits:
  - Heading + truth value
  - `duplicate_of` / `superseded_by` metadata links
  - Defeater links (scans every justification's `outlist` for entries whose `metadata.defeats_node == node_id`)
  - The belief's own text
  - **Antecedent links** split into "active" (from the designated `supporting_justification`) and "other" (from remaining justifications)
  - **Dependent links** (reverse edges)

  Every cross-reference is resolved through `node_to_page`; if the target node isn't on any page, the ID is rendered as plain text (no link).

- **`_format_beliefs_for_prompt(node_ids, node_details)`** → `str`
  A simpler flat rendering (no defeaters, no duplicate/superseded links) optimized for token efficiency in the LLM prompt. Only includes the active antecedents and dependents.

- **`generate_wiki_page(topic, node_ids, node_details, model, timeout)`** → `str`
  Thin wrapper: formats beliefs, interpolates the prompt, calls `invoke_model`.

- **`_linkify(content, current_page, node_to_page, all_ids)`** → `str`
  Post-processing pass. Scans the rendered page text for bare node IDs (using a word-boundary regex that respects `-`) and converts them to `[id](page#id)` Markdown links. Sorts IDs by length descending so that `foo-bar` is matched before `foo`. Skips IDs that are already linked or belong to the current page.

### Function (public)

- **`build_wiki(node_details, groups, output_dir, model="", timeout=300, parallel=0)`** → `dict`
  The entry point. Contract:

  | Parameter | Type | Notes |
  |---|---|---|
  | `node_details` | `dict[str, dict]` | Keyed by node ID; each value is a `show_node`-shaped dict with at minimum `truth_value`, `text`, `justifications`, `dependents`. |
  | `groups` | `dict[str, list[str]]` | Topic/cluster label → list of node IDs. |
  | `output_dir` | `str` | Created with `makedirs` if absent. |
  | `model` | `str` | Empty string = no LLM; non-empty = synthesize. |
  | `timeout` | `int` | Per-call LLM timeout (seconds). |
  | `parallel` | `int` | 0 = sequential; >0 = ThreadPoolExecutor workers. |

  Returns `{"output_dir": ..., "pages": int, "total_nodes": int}`.

## Patterns

- **Two-mode rendering (strategy).** Each page is either LLM-generated prose *or* a deterministic per-node dump. The choice is made once at the top of the write loop via `if label in generated_content`. This lets the LLM path fail gracefully (falls back to the deterministic path) without a separate code path.

- **Parallel-then-sequential fallback.** The `parallel` parameter controls whether `ThreadPoolExecutor` is used. The sequential path is the `elif model:` branch. Both share the same `generate_wiki_page` call, so the unit of work is identical.

- **Deferred import.** `invoke_model` and `ThreadPoolExecutor` are imported inside functions, not at module top level. This keeps the module importable in environments where the LLM SDK or the standard library's `concurrent.futures` (rarely an issue, but) might not be available, and avoids a circular-import risk with `reasons.llm`.

- **Cross-reference resolution table.** The `node_to_page` dict is built once and passed to every rendering function. This is a classic "adjacency lookup" pattern — O(1) page resolution instead of re-scanning groups.

- **Slug collision resolution.** The `while slug in used_slugs` loop appends `-2`, `-3`, etc. Combined with `_RESERVED_SLUGS`, this guarantees unique filenames without raising.

## Dependencies

```
build_wiki.py
├── stdlib:  os, re, sys
├── intra-pkg: .llm → invoke_model()          (lazy, inside generate_wiki_page)
└── stdlib:  concurrent.futures               (lazy, inside build_wiki)
```

**Imported by:**

| Consumer | Usage |
|---|---|
| `reasons/api.py` | High-level API surface; exposes `build_wiki` as a REST/MCP endpoint. |
| `reasons_lib/build_wiki.py` | Library mirror (the `reasons_lib` tree is a vendored/parallel copy of `reasons`). |
| `tests/test_build_wiki.py` | Unit/integration tests. |

There are **no** imports of `build_wiki` from the other `reasons/*.py` modules (e.g., `cluster.py`, `compact.py`), so it sits at a leaf of the dependency graph: it consumes the belief-network data model but is consumed only by the API layer and tests.

## Flow

```
build_wiki()
│
├─ 1. os.makedirs(output_dir)
│
├─ 2. Slug allocation
│     for each group label:
│       slug = _page_name(label)
│       if slug in _RESERVED_SLUGS → slug += "-topic"
│       deduplicate → slug-2, slug-3, …
│       label_to_file[label] = slug + ".md"
│
├─ 3. Build node_to_page: {node_id → "slug.md"}
│
├─ 4. Write index.md
│     Sorted by (-belief_count, label) so largest topics first.
│     Each row links to the group page.
│
├─ 5. LLM generation (if model is set)
│     ├─ parallel > 0 → ThreadPoolExecutor
│     │   Each future: generate_wiki_page() → invoke_model()
│     │   Progress logged to stderr: "Generated <label> (n/total)"
│     │   Failures caught → logged as WARN, label skipped
│     └─ else → sequential loop, same try/except per label
│
├─ 6. Write per-group pages
│     for each (label, nids):
│       page = "# {label}\n"
│       page += "[Back to index](index.md)\n"
│       if LLM content exists for this label:
│           page += generated prose
│       else:
│           for each nid (sorted):
│               page += _format_node(nid, detail, node_to_page, all_details)
│       page = _linkify(page, …)
│       write to output_dir/slug.md
│
└─ 7. return {"output_dir", "pages", "total_nodes"}
```

The data transformation pipeline per node is:

```
node_detail dict
  → _format_node()        (structured markdown with links)
  → _linkify()            (catches any bare IDs in LLM prose)
  → written to file
```

## Invariants

1. **Unique filenames.** The slug-deduplication loop guarantees no two groups produce the same `.md` file. The reserved-slug check prevents a topic named "index" from overwriting the index page.

2. **Every node appears on exactly one page.** A node ID is a key in `node_to_page` exactly once (built from `groups`). If a node ID appears in two groups, the last write wins in `node_to_page` — but the code does not guard against this; the caller is responsible for non-overlapping groups.

3. **`supporting_justification` index is bounds-checked.** Both `_format_node` and `_format_beliefs_for_prompt` guard with `0 <= si < len(justifications)` before using it. If it's out of range, the code falls through to the "all antecedents" path.

4. **Linkify idempotence.** `_linkify` skips IDs that already appear inside `[id](...)` syntax (checks `if "[" + nid + "](" in content`), so running it twice won't double-link.

5. **Node ID matching is token-based, not substring-based.** `_assign_topics` splits on `[-._:]` and requires an exact token match against the topic label, so topic `"ml"` won't accidentally match node `"machine-learning-models"`.

## Error Handling

The philosophy here is **best-effort with stderr diagnostics**:

- **LLM generation failure** (network timeout, model error, bad response): caught by `except Exception as e` in both the parallel and sequential paths. The error is printed to `sys.stderr` as `WARN: <label> failed: <e>`. The affected group falls back to the deterministic per-node rendering. No exception propagates to the caller.

- **Missing node detail** (`node_details.get(nid)` returns `None`): silently skipped in both `_format_node` (caller guards) and `_format_beliefs_for_prompt` (`if not detail: continue`).

- **Missing cross-reference target** (a dependent or antecedent node that isn't in any group): the ID is rendered as plain text rather than a link. No warning is emitted.

- **`_assign_topics` unmatched nodes**: silently placed in `"Other"`.

- **No validation of `node_details` schema.** The code assumes `truth_value`, `text`, `justifications`, and `dependents` keys exist. A missing key will raise `KeyError` in `_format_node` (specifically `node_detail['truth_value']` and `node_detail['text']` are direct-indexed, not `.get()`-ed). This is the one spot that will crash rather than degrade gracefully.

- **No file-write error handling.** `open(..., "w")` and `f.write(...)` are unguarded. A disk-full or permissions error will propagate as an unhandled `OSError`.

## Topics to Explore

- [file] `reasons/cluster.py` — Produces the `groups` dict (semantic clusters) that feeds `build_wiki`; understanding how clusters are labeled explains what topic strings land in `groups`.
- [function] `reasons/llm.py:invoke_model` — The single LLM call boundary; its error semantics (does it raise or return an error string?) determine how `generate_wiki_page` failures manifest.
- [file] `reasons/api.py` — Shows how `build_wiki` is exposed to callers (REST/MCP), what validation is applied to `node_details` and `groups` before the call, and how the return dict is serialized.
- [file] `tests/test_build_wiki.py` — Covers the slug-collision, parallel-vs-sequential, defeater-linking, and `_linkify` idempotence paths; reading it reveals the contractual edge cases.
- [general] `tms-node-detail-shape` — The exact schema of a `show_node` dict (what `justifications[].outlist`, `supporting_justification`, `metadata.defeats_node`, `metadata.duplicate_of` look like) is the implicit contract this renderer depends on; tracing it back to `reasons/network.py` or `reasons/storage.py` clarifies the data model.

## Beliefs

- `build-wiki-produces-static-markdown` — `build_wiki` writes exclusively to the local filesystem (`output_dir`); it has no network I/O except the optional LLM call, and no database access.
- `llm-failure-falls-back-to-deterministic-rendering` — If `generate_wiki_page` raises for a given group, that group is rendered via `_format_node` per-node instead; the wiki is still complete.
- `node-to-page-is-built-before-rendering` — The `node_to_page` mapping is fully constructed before any page is written, so every cross-reference link in every page is resolved against the complete set of groups, not a partial one.
- `linkify-skips-already-linked-ids` — `_linkify` checks for the `[id](` prefix before substituting, making it safe to run on content that already contains hand-written Markdown links; it will not produce `[id](page#id](page#id)`.
- `supporting-justification-is-bounds-checked` — Both `_format_node` and `_format_beliefs_for_prompt` guard `supporting_justification` with a `0 <= si < len(justifications)` check before indexing, preventing `IndexError` on malformed node details.
