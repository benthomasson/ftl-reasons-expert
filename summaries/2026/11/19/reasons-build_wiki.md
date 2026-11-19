---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/build_wiki.py

# `reasons/build_wiki.py` — Static Wiki Generator

## Purpose

This module owns the **"publish the belief network as a browsable static site"** responsibility. It takes the current state of a Truth Maintenance System — `{node_id: show_node dict}` — plus a grouping of those nodes (either word-frequency topics or semantic clusters produced elsewhere) and emits a directory of interlinked markdown files:

- `index.md` — a table of contents listing each group with a belief count.
- `<slug>.md` — one page per group, containing either (a) an LLM-synthesized narrative or (b) a plain per-belief render (`_format_node`) when no model is supplied or the LLM call fails.

It deliberately does **not** do its own grouping: `_assign_topics` is a helper, but the caller supplies the `groups` dict. It also does not fetch nodes — it receives them via `node_details` and returns a small stats dict.

---

## Key Components

### Module-level constants

| Name | Purpose |
|------|---------|
| `_TOPIC_STOP_WORDS` | A large `set` of English function words. **Note: unused inside this module.** Likely a vestigial/shared constant or intended for external importers. Worth flagging in code review as dead code. |
| `WIKI_PAGE_PROMPT` | `str.format` template. Two placeholders: `{topic}` and `{beliefs}`. Instructs the model to write encyclopedic prose, embed belief IDs in parens, note dependency relationships, and to *not* emit a title. |
| `_RESERVED_SLUGS = {"index"}` | Slugs that would collide with `index.md`; a topic literally named "index" is rewritten to `index-topic`. |

### Helper functions

**`_assign_topics(node_ids, topics)`** — not called by `build_wiki`, exposed for callers.
- Splits each `node_id` on `[-._:]` and filters tokens of length ≤ 2.
- First token that matches a topic label wins (`break` in the inner loop).
- Unmatched nodes go to a synthetic `"Other"` bucket.
- Empty buckets are dropped at the end (`{k: v for k, v in groups.items() if v}`), so the returned dict size can differ from `len(topics)`.

**`_page_name(label)`** — sanitizer for filenames.
- `label.lower()` → collapse every run of non-`[a-z0-9]` to `-` → strip leading/trailing `-`.
- Empty result falls back to `"other"`. Note this means two distinct labels that both sanitize to empty would collide — resolved later by the collision loop in `build_wiki`.

**`_format_node(node_id, node_detail, node_to_page, all_details=None)`** — the fallback renderer (and the only renderer when `model=""`).
Produce order:
1. `### {node_id}` heading.
2. `**Status:** {truth_value}` (e.g. `IN`, `OUT`).
3. `**Duplicate of:**` / `**Superseded by:**` — from `metadata["duplicate_of"]` / `metadata["superseded_by"]`, each resolved into an in-wiki link if the target is a known node.
4. **Defeater scan** (guarded by `if all_details:`): iterate `node_detail["justifications"]` → each `j["outlist"]` → look up `all_details[o]` → keep only those whose `metadata["defeats_node"] == node_id`. Renders as `**Defeated by:** <link> (<defeater_type>, <defeat_reason_type>)`. This is a *reverse* lookup through the outlist — the file that "is" a defeater's target is discovered by scanning outlists of the target's justifiers.
5. Body text (`node_detail["text"]`).
6. **Dependency split** — if `supporting_justification` indexes a valid justification, antecedents from *that* justification get labeled `**Depends on (active):**`; antecedents unique to other justifications get `**Depends on (other):**`. Otherwise all antecedents collapse into `**Depends on:**`.
7. `**Supports:**` from `node_detail["dependents"]`.
- Every cross-reference (`a`, `d`, `d_id`, `new_id`, `canonical_id`) is turned into `[id](page#id)` if the target page is known, else rendered as plain text. Links are consistently anchored with `#node_id`.

**`_format_beliefs_for_prompt(node_ids, node_details)`** — compact text block for the LLM. Unlike `_format_node`, it does **not** walk outlists or metadata; it only emits `### id`, `Status:`, `Text:`, `Depends on:` (deduped union of antecedents across justifications, but prefixed to the *single* designated justification's antecedents when `supporting_justification` is valid), and `Supports:`.

**`generate_wiki_page(topic, node_ids, node_details, model, timeout)`** — thin wrapper. Imports `invoke_model` **lazily** inside the function (avoids importing the LLM stack for callers that only render statically). Returns the model's string.

**`_linkify(content, current_page, node_to_page, all_ids)`** — post-processing pass over a *whole rendered page*.
- Iterates IDs sorted by **length descending** so longer IDs (which may contain shorter ones as substrings) get linked first.
- Skips IDs whose page equals `current_page` (intra-page references are left as literal text — anchors work, but they'd be redundant, and self-headings already exist).
- Skips IDs not present in `content` (fast path).
- Skips IDs already appearing as `[nid](` (double-link guard — string-prefix match, not a regex; fragile if the ID is linked in a different format elsewhere, e.g. `[id](page#id)` inside a compound link).
- Uses `(?<![a-z0-9\-])` / `(?![a-z0-9\-])` lookarounds as a word-boundary-ish check. Does **not** account for uppercase or `_` as boundary characters.

### Main entry point

```python
build_wiki(node_details, groups, output_dir, model="", timeout=300, parallel=0)
```

Returns `{"output_dir": str, "pages": int, "total_nodes": int}`.

**Contract:**
- `node_details: dict[str, dict]` — keys `truth_value`, `text`, `justifications` (list of `{antecedents, outlist}`), `supporting_justification` (optional int), `dependents` (optional list), `metadata` (optional dict).
- `groups: dict[str, list[str]]` — group label → node IDs. **The same node ID appearing in two groups is silently resolved** (last write wins in `node_to_page`) — see Invariants.
- `model == ""` **and** `parallel > 0` → no LLM (parallel branch guarded on both). `model != ""` with `parallel == 0` → sequential with progress messages.

---

## Patterns

- **Two-tier rendering**: "LLM or fallback". The generator writes to `generated_content[label]` only on success; the write loop then uses `if label in generated_content: use it, else: iterate _format_node`. This is a clean *producer/consumer split* that keeps the writing loop deterministic.
- **Two-phase output**: (1) generate all LLM content (possibly in parallel), (2) write files. This means failures in one page cannot corrupt others, and stderr progress is naturally interleaved with generation, not with disk I/O.
- **Slug registry as collision oracle**: `used_slugs: dict[str, str]` maps slug → label. The `while slug in used_slugs: slug = f"{slug}-2"` loop is a naive deduplicator. Multiple collisions yield `foo`, `foo-2`, `foo-2-2`, … (not `foo-3`) — a known quirk.
- **Lazy imports** for optional deps: `ThreadPoolExecutor` imported inside the `parallel > 0` branch; `invoke_model` imported inside `generate_wiki_page`. Keeps static-wiki generation dependency-free.
- **Linkification as a post-pass** rather than during rendering: centralizes anchor-format decisions in one place, but means the renderer must emit raw IDs and the linkifier has to avoid re-linking already-linkified text.
- **Anchor scheme = node ID**: `page#node_id`. Because IDs are the anchor names, no slugification occurs on nodes — the ID is assumed URL-safe.

---

## Dependencies

**Imports (top of file):** `os`, `re`, `sys`.

**Lazy/conditional imports:**
- `.llm.invoke_model` — only when `model` is truthy.
- `concurrent.futures.ThreadPoolExecutor`, `as_completed` — only when `model and parallel > 0`.

**Imported by:**
- `reasons/api.py` — presumably wraps `build_wiki` in the public API.
- `reasons_lib/build_wiki.py` — mirror package (the repo appears to ship both `reasons/` and `reasons_lib/` with a parallel tree; maintainers likely keep them in sync).
- `tests/test_build_wiki.py` — exercises it directly.

Notably it depends on `node_details` entries being *already evaluated* (`truth_value` is prepopulated) — this module never queries storage or a network.

---

## Flow

```
build_wiki(node_details, groups, output_dir, model, timeout, parallel)
│
├─ os.makedirs(output_dir, exist_ok=True)
│
├─ Slug assignment (per label, deterministic given dict order):
│     slug = _page_name(label)
│     if slug in _RESERVED_SLUGS: slug += "-topic"
│     while slug in used_slugs: slug += "-2"
│     → label_to_file[label] = slug + ".md"
│
├─ node_to_page: {node_id → "slug.md"} built by walking groups in order
│
├─ Write index.md:
│     "# Belief Wiki" + table header
│     rows sorted by (-len(nodes), label)
│     footer: "*N beliefs across M pages*"
│
├─ LLM phase (only if model):
│     parallel > 0 → ThreadPoolExecutor(max_workers=parallel)
│         submit _gen(label, nids) → (label, invoke_model(...))
│         as_completed: store in generated_content, print "Generated …"
│         except → print "WARN: … failed …" (no re-raise)
│     else (sequential):
│         for each group: print "Generating …", store or WARN
│
└─ Write each page:
      header "# {label}" + "[Back to index](index.md)"
      if generated_content[label]: append raw LLM output
      else: for nid in sorted(nids): _format_node(...,
             all_details=node_details)
      page_text = _linkify(page_text, page_file, node_to_page,
                           node_details.keys())
      write to output_dir/page_file
```

Ordering that matters:
1. `node_to_page` must be complete **before** `_format_node` is called, because links are resolved during rendering.
2. `_linkify` runs **after** both the LLM content and fallback render are joined, so it can retroactively link IDs mentioned in prose.
3. Index is written **before** pages — a partial run leaves a valid `index.md`.

---

## Invariants

- **Filename uniqueness**: guaranteed by `used_slugs` + `_RESERVED_SLUGS` + `-topic` rewrite. Two labels that sanitize identically (`"SQL/DB"` and `"sql.db"`) end up as separate files; the second gets `-2`.
- **Every group has at least a page file** if it exists in `groups` — even empty groups get a stub (though `_assign_topics` filters out empties, external callers might pass them).
- **Node IDs appear in exactly one page**: `node_to_page` is a plain dict; if a node is in group A *and* group B, the *later-iterated* group reassigns it. `groups.items()` order therefore affects which page a duplicate appears on. This is an **undocumented** but real invariant the caller must be aware of.
- **`supporting_justification` is treated as an index into `justifications`**, not as an object reference. If it is out of range, the code silently falls back to the "all antecedents are active" branch (`else:` at the bottom of the `_format_node` / `_format_beliefs_for_prompt` blocks).
- **Defeaters are discovered by reverse scan**, not by a direct `defeaters` field. The precondition is that the *defeater node's* `metadata["defeats_node"]` equals the target — otherwise the `**Defeated by:**` line is not rendered even if the outlist entry exists.
- **Anchor format is fixed**: `#<node_id>` for every node. IDs must be URL-fragment-safe; no sanitization is applied.
- **LLM output is trusted verbatim** — no validation that it respects the "no title" instruction. If the model emits `# Title`, the page will have two `#` headings.
- **`index` is a reserved page name**; no group is allowed to overwrite it.

---

## Error Handling

- **LLM failures are swallowed** at both call sites: `except Exception as e: print(f"WARN: {label} failed: {e}", file=sys.stderr)`. The group's page then falls through to the plain `_format_node` path. This is *intentional* — one flaky model call never aborts a wiki build.
- **File I/O errors are NOT caught** — `open(..., "w")` failures propagate. So a read-only `output_dir` aborts mid-run.
- **`node_details.get(nid)` returning `None`** (missing node referenced by a group) is silently skipped in the write loop (`if detail:`) but **not** in `_format_beliefs_for_prompt` (it `continue`s), meaning the LLM sees fewer beliefs than `groups` claims. There is no validation that group member IDs exist in `node_details` — a real footgun given `build_wiki` returns `total_nodes = sum(len(nids))`, which counts the missing ones.
- **No `KeyError` protection** on `node_detail['truth_value']` or `['text']` — malformed node dicts crash the renderer.
- **Parallel path exceptions** are caught per-future via `future.result()` inside a `try`, so a single failure doesn't cancel siblings — matching the sequential behavior.
- **`_linkify` is safe against catastrophic backtracking**: `re.escape(nid)` + lookarounds (no nested quantifiers). Cost is O(number of ids × content length).

---

## Topics to Explore

- [file] `reasons/cluster.py` — Where `groups` passed to `build_wiki` is produced; understanding it clarifies topic vs. cluster trade-offs.
- [file] `reasons/llm.py` — Backs `invoke_model`; check whether `timeout` and `model` are the only knobs and whether retries exist.
- [function] `reasons/api.py:build_wiki` — How the public API wraps this and where `model`/`parallel` defaults are set.
- [file] `tests/test_build_wiki.py` — Confirms the actual contract (fallback rendering, linkification, and slug dedupe edge cases) that this file just implies.
- [general] `reasons-lib-mirror` — Why both `reasons/` and `reasons_lib/` contain this file; likely a packaging shim but worth confirming how they stay in sync.

## Beliefs

- `build-wiki-does-not-perform-grouping` — `build_wiki` receives the `groups` dict fully formed; it never calls `_assign_topics` itself.
- `build-wiki-swallows-llm-errors` — LLM generation failures in `build_wiki` are caught, logged to stderr as `WARN`, and the page falls back to raw markdown rendering.
- `build-wiki-parallel-requires-model` — The parallel branch is only entered when **both** `model` is truthy and `parallel > 0`; `parallel > 0` with `model=""` degrades to the static renderer.
- `node-to-page-maps-node-ids-to-filenames` — `node_to_page` is a `{node_id: "<slug>.md"}` dict built before rendering and reused by `_format_node` and `_linkify` for cross-reference resolution.
- `linkify-sorts-ids-by-length-descending` — `_linkify` processes IDs longest-first so that IDs which are substrings of other IDs are linked after longer matches.
- `assign-topics-parses-id-segments` — `_assign_topics` splits node IDs on `[-._:]`, drops tokens of length ≤ 2, and matches the first segment against the topic set.
- `reserved-slug-index-rewrites-to-index-topic` — A group label whose slug is `index` (the reserved slug) is rewritten to `index-topic` before file I/O.
