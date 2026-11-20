---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: build_wiki in reasons/build_wiki.py

## `build_wiki`

### Purpose
Orchestrator for emitting a static, interlinked markdown wiki from a belief network (Truth Maintenance System) snapshot. It writes a top-level `index.md` plus one page per topic/cluster group. Each page body is either:

- LLM-synthesized narrative (when `model` is non-empty), or
- A concatenation of raw per-node markdown blocks produced by `_format_node`.

It exists to make the belief graph browsable as ordinary files (e.g., served from a static site or committed to a repo), without requiring a running service.

### Contract

**Preconditions (assumed, not enforced):**
- `groups` is a dict; each *label* is a unique string; each list of node IDs is iterable.
- `node_details[nid]` for any node that appears in a group provides at least `"truth_value"` and `"text"` keys (accessed directly in `_format_node`), and optionally `"metadata"`, `"justifications"`, `"supporting_justification"`, `"dependents"`.
- Each node ID appears in **at most one** group. If violated, the node is rendered on multiple pages while `node_to_page` (last-writer-wins) points cross-links to whichever group was processed last.
- `output_dir` (or its parents) is writable.
- If `model` is non-empty, `generate_wiki_page`/`invoke_model` accepts that model string.

**Postconditions:**
- `output_dir` exists.
- `output_dir/index.md` is always written (even if `groups` is empty).
- Exactly one `*.md` file per label in `groups` (filename decided by `_page_name` + collision resolution).
- Returned dict reports the directory, page count (= `len(groups)`), and `total_nodes` (= sum of list lengths, **not** unique node count).

**Invariants during the run:**
- `used_slugs` maps every emitted slug back to its label; `label_to_file` maps every label to its `*.md` filename.
- `node_to_page` maps every node ID encountered in any group to a single page filename.
- Rendering of nodes is deterministic (nodes are sorted before rendering).

### Parameters

| Name | Type | Meaning / edge cases |
|---|---|---|
| `node_details` | `dict[node_id, dict]` | The "show_node" payload for each belief. Missing keys cause `_format_node` to raise. Nodes present here but absent from `groups` are never rendered — but *are* eligible for cross-link substitution because `_linkify` is given `node_details.keys()`. |
| `groups` | `dict[label, list[node_id]]` | Grouping produced by upstream (`_assign_topics` or semantic clustering). Empty dict is legal. Order of keys controls sequential generation order and page-write order (but not index order). |
| `output_dir` | `str` / path | Created via `os.makedirs(..., exist_ok=True)`. Overwrites any pre-existing `index.md` or slug-matching pages. |
| `model` | `str` | Empty string ⇒ **no LLM**; raw node rendering path. Non-empty ⇒ LLM path. |
| `timeout` | `int` (seconds) | Passed straight to `generate_wiki_page`/`invoke_model`. |
| `parallel` | `int ≥ 0` | `> 0` **and** `model` non-empty ⇒ thread pool. `0` or negative with non-empty `model` ⇒ sequential. Ignored when `model == ""`. |

### Return Value
```python
{
    "output_dir": str,
    "pages": int,       # len(groups)
    "total_nodes": int, # sum(len(nids) for nids in groups.values())
}
```
No error indicator is returned. **Caller obligations:** must check `stderr` for `WARN:` lines and re-inspect the output directory if a partial result is possible — the function does not raise on LLM failures, so a returned dict does **not** imply every page was LLM-generated.

### Algorithm

1. **Directory + slug assignment**
   - `os.makedirs(output_dir, exist_ok=True)`.
   - For each label (dict insertion order), compute `_page_name(label)` (lowercase, non-alnum → `-`, strip `-`, fallback `"other"`).
   - If slug ∈ `_RESERVED_SLUGS` (`{"index"}`), append `-topic`.
   - While the slug collides with a previous one, append `-2`. Note the suffix is appended cumulatively, so three collisions look like `foo`, `foo-2`, `foo-2-2` — **not** `foo-3`.
   - Record `label_to_file[label] = slug + ".md"`.

2. **Reverse map**
   - `node_to_page[nid] = label_to_file[label]` for every nid in every group. Later groups overwrite earlier entries.

3. **Index page**
   - Header `# Belief Wiki`, a two-column table.
   - Rows sorted by `(-len(groups[label]), label)` — **descending size, then alphabetical**.
   - Footer line `*{total} beliefs across {pages} pages*`.
   - Written immediately (before any LLM call).

4. **LLM generation (optional)**
   - `if model and parallel > 0`: build a `ThreadPoolExecutor(max_workers=parallel)`, submit `_gen` per group, consume via `as_completed`, print progress to `stderr` (`Generated {label} ({done}/{total})`). On exception: print `WARN: ... failed: {e}` and continue — that label is simply absent from `generated_content`.
   - `elif model`: sequential loop, prints `Generating {label} ({i}/{total})...` before each call; same swallow-on-error semantics.
   - Otherwise: `generated_content` stays empty ⇒ all pages use raw rendering.

5. **Page emission** (deterministic order matching `groups`)
   - Header `# {label}`, blank line, `[Back to index](index.md)`.
   - If the label has generated content, append it verbatim (then a blank line).
   - Else, for each `nid` in **sorted** order, if `node_details.get(nid)` is non-empty, append `_format_node(nid, detail, node_to_page, all_details=node_details)`.
   - Join with `\n`, then run `_linkify(page_text, page_file, node_to_page, node_details.keys())` to replace bare node IDs with `[id](target.md#id)` cross-links (skipping same-page links and already-linked IDs). Note that the `_linkify` universe includes IDs from `node_details` that never appear in any group; those are skipped because `node_to_page.get` returns `None`.
   - Write the file.

6. Return the summary dict.

### Side Effects
- Filesystem: creates `output_dir`; writes/overwrites `index.md` and every `*.md` page.
- Network/LLM: via `generate_wiki_page → invoke_model` when `model` is set — one call per group (sequential or parallelized up to `parallel` threads).
- Stderr: progress and `WARN:` lines.
- Threads: `ThreadPoolExecutor` workers if `parallel > 0`.
- No mutation of `node_details` or `groups` — they are read-only inputs.

### Error Handling
- **Swallowed:** any exception from `generate_wiki_page` in either branch — logged as `WARN:` to stderr. The affected label simply falls back to raw rendering on the next loop.
- **Propagated:** `OSError`/`PermissionError` from `os.makedirs` or `open(...).write(...)`; `KeyError`/`TypeError` from `_format_node` if `node_detail["truth_value"]` / `["text"]` are missing; regex/search errors in `_linkify`. A failure in step 5 leaves `index.md` and any pages written before the error on disk (partial state).
- **Not detected:** duplicates across groups (silently reflected in `total_nodes` and index counts); labels colliding with reserved files after `-topic` suffixing (handled via used-slug loop); nodes in `groups` whose ID is absent from `node_details` (skipped silently by `_format_node` guard in the loop, but still counted in `total`).

### Usage Patterns
Typical call from a CLI or driver:

```python
groups = _assign_topics(node_ids, topics)             # or semantic clustering
summary = build_wiki(node_details, groups, "out/wiki",
                      model="gpt-4o-mini", parallel=4)
```

Callers should:
- Add the output directory to `.gitignore` if regenerated per run.
- Treat stderr `WARN:` output as a signal that some pages are un-synthesized.
- Not assume `total_nodes` is a unique count — it is a *sum of group sizes*.

### Dependencies
- Standard library: `os`, `re`, `sys`, `concurrent.futures.ThreadPoolExecutor/as_completed`.
- Intra-module: `_page_name`, `_format_node`, `_linkify`, `_RESERVED_SLUGS`, `generate_wiki_page`.
- Transitive: `reasons.llm.invoke_model` (only touched when `model != ""`).

---

## Topics to Explore

- [file] `tests/test_build_wiki.py` — Confirms expected file names, index contents, and LLM/raw fallbacks; the authoritative regression suite for this function.
- [function] `reasons/build_wiki.py:_format_node` — Renders a single belief as markdown; the primary code path when `model==""` and the fallback path when LLM generation fails.
- [function] `reasons/build_wiki.py:_linkify` — Deterministic cross-URL linking of node IDs; governs the inter-page navigation guarantees of the wiki.
- [function] `reasons/build_wiki.py:_page_name` — Slug derivation and collision surface; understanding it is prerequisite to prediction of filenames.
- [general] `llm-generation-fallback` — How partial LLM failures degrade the wiki into raw-node rendering, and how callers should detect it.

## Beliefs

- `build-wiki-produces-index-and-per-group-pages` — Writes exactly one `index.md` plus one markdown file per label in `groups`.
- `build-wiki-swallows-llm-exceptions` — Exceptions from `generate_wiki_page` are caught, logged to stderr as `WARN:`, and the affected label falls back to raw-node rendering.
- `slug-collision-appends-2-repeatedly` — The collision loop appends `-2` cumulatively (`foo`, `foo-2`, `foo-2-2`), not an incremental counter.
- `build-wiki-requires-node-details-truth-value-and-text` — `_format_node` directly indexes `node_detail["truth_value"]` and `node_detail["text"]`, so a missing key raises inside `build_wiki`.
- `node-to-page-mapping-is-last-writer-wins` — If a node ID appears in multiple groups, `node_to_page[nid]` points to the last group's page even though the node is rendered on every containing page.
