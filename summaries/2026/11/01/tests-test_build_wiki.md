---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_build_wiki.py

# `tests/test_build_wiki.py` — Explanation

## 1. Purpose

This file is the test suite for the **`reasons.build_wiki`** module, which is responsible for generating a human-readable Markdown wiki from the belief graph stored in the reasons database. The "build-wiki" command takes a collection of belief nodes (with their justifications, dependents, truth values, and metadata) and renders them into a set of interlinked Markdown pages, organized by topic.

The test file owns responsibility for verifying:
- Correct topic assignment and grouping of nodes
- Markdown formatting of individual belief nodes (including cross-references, duplicates, superseded/superseding relationships, and defeater annotations)
- File generation logic (index page, topic pages, slug collision handling)
- LLM-assisted page generation (and its fallback behavior)
- End-to-end CLI invocation

## 2. Key Components

### Helper: `run_cli(*args, db_path=None)`
A local utility (lines 14–29) that simulates calling `reasons.cli.main()` with arbitrary arguments. It:
- Constructs an `argv` list starting with `"reasons"`
- Optionally prepends `--db <path>`
- Patches `sys.argv`, `sys.stdout`, `sys.stderr` via `unittest.mock.patch`
- Catches `SystemExit` to extract the exit code
- Returns a tuple `(stdout_str, stderr_str, exit_code)`

This is the standard "test the CLI without spawning a subprocess" idiom.

### Fixture: `db(tmp_path)`
Creates a temporary SQLite database seeded with 5 nodes spanning 3 topics (caching, testing, deploy) and one justification edge (`deploy-ci-pipeline` ← `testing-unit-coverage`). Used by `TestBuildWikiApi`, `TestBuildWikiCli`.

### Test Classes

| Class | Focus |
|-------|-------|
| `TestAssignTopics` | `_assign_topics` — bucketing node IDs into topic groups, with "Other" fallback |
| `TestPageName` | `_page_name` — slugifying topic names into valid filenames |
| `TestFormatNode` | `_format_node` — rendering a single belief as Markdown with cross-links, metadata (duplicate_of, superseded_by, defeated_by) |
| `TestBuildWiki` | `build_wiki` — the core orchestration: file creation, index generation, slug collision handling |
| `TestBuildWikiApi` | `api.build_wiki` — integration: real DB → wiki output, status filtering, empty DB |
| `TestBuildWikiCli` | CLI entry point: `--help`, `-o` output, `--status` filter |
| `TestLinkify` | `_linkify` — post-processing pass that converts bare node IDs into Markdown links |
| `TestFormatBeliefsForPrompt` | `_format_beliefs_for_prompt` — building the LLM prompt text |
| `TestGenerateWikiPage` | `generate_wiki_page` — verifying it calls `reasons.llm.invoke_model` |
| `TestBuildWikiLlm` | LLM mode in `build_wiki`: generation, parallelism, fallback on failure |

### Key functions under test (from `reasons.build_wiki`)

- **`_assign_topics(node_ids, topics) → dict[str, list[str]]`** — Maps each node to the first topic whose key is a prefix of the node ID. Unmatched nodes go to `"Other"`.
- **`_page_name(topic) → str`** — Slugifies a topic string into a filename-safe token. Falls back to `"other"` if the result is empty.
- **`_format_node(node_id, detail, node_to_page, all_details=None) → str`** — Renders one belief block. Produces `### <id>`, status, text, cross-references (as `[id](page.md#id)`), and metadata lines (Duplicate of, Superseded by, Defeated by).
- **`_linkify(content, current_page, node_to_page, node_ids) → str`** — Regex/iteration pass that replaces bare node IDs with Markdown links. Skips same-page references and already-linked text. Sorts by length (longest first) to avoid partial matches.
- **`build_wiki(details, groups, output_dir, model=None, parallel=1) → dict`** — The orchestrator. Writes one `.md` per topic + an `index.md`. Returns `{"pages": int, "total_nodes": int}`.
- **`generate_wiki_page(topic, node_ids, details, model, max_tokens) → str`** — Builds a prompt via `_format_beliefs_for_prompt` and calls `reasons.llm.invoke_model`.

## 3. Patterns

- **Layered testing strategy**: Unit tests for pure helpers (`_assign_topics`, `_page_name`, `_format_node`), integration tests for the API (`api.build_wiki` with a real DB), and CLI smoke tests (`run_cli`). This mirrors the project's general convention of testing at three abstraction levels.

- **Mock-based LLM isolation**: All LLM calls are patched (`patch("reasons.llm.invoke_model", ...)`, `patch("reasons.build_wiki.generate_wiki_page", ...)`). No real network calls occur. The `side_effect=RuntimeError` test in `TestBuildWikiLlm` explicitly verifies the **graceful fallback** path.

- **`tmp_path` fixture**: Every test that writes files uses pytest's built-in `tmp_path`, ensuring no cross-test contamination and no cleanup logic in the tests themselves.

- **CLI testing via `StringIO` + `patch`**: The `run_cli` helper is a well-known pattern for testing `click`/`argparse`-based CLIs in-process. It avoids subprocess overhead and lets tests assert on both stdout and exit code.

- **Table-driven style within classes**: Each `Test*` class groups related assertions. For example, `TestFormatNode` has 11 methods each testing one metadata/relationship scenario (basic, cross-refs, duplicate, superseded, defeated-by with/without page, defeated-by with reason type, None metadata).

- **Slug collision testing**: `test_index_slug_collision` and `test_duplicate_slug_collision` specifically guard against the edge case where a topic name produces the same slug as `index` or another topic — a real-world bug class in file generation.

## 4. Dependencies

### This file imports:
| Import | Role |
|--------|------|
| `reasons.api` | Public API surface (`api.add_node`, `api.build_wiki`, `api.retract_node`) |
| `reasons.build_wiki` | Under-test module: `_assign_topics`, `_format_beliefs_for_prompt`, `_format_node`, `_linkify`, `_page_name`, `build_wiki`, `generate_wiki_page` |
| `reasons.cli` (inside `run_cli`) | CLI entry point `main()` |
| `reasons.llm` (via patch target) | `invoke_model` — the LLM call abstraction |
| `pytest` | Test runner, `tmp_path` fixture |
| `unittest.mock.patch` | Mocking for sys.argv/stdout and LLM functions |

### What depends on this file:
Nothing imports test files. It is a leaf in the dependency graph, consumed only by the test runner (pytest).

### Test isolation notes:
- `reasons.build_wiki` is the system under test; `reasons.llm` is its external dependency (mocked).
- `reasons.api` acts as a thin facade over `reasons.build_wiki` + storage.
- The `db` fixture depends on `reasons.api.add_node` and the underlying `reasons.storage` / `reasons.pg` layer.

## 5. Flow

The test suite doesn't have a single "flow" — it exercises many independent code paths. However, the **primary execution path** it validates (the `build_wiki` call) is:

```
api.build_wiki(output_dir, db_path, status=None, model=None)
  │
  ├─ Load all nodes from DB (filter by status if provided)
  ├─ Classify topics from node IDs (prefix-based)
  ├─ _assign_topics() → groups: {topic: [node_ids]}
  ├─ For each topic:
  │    ├─ If model is set → generate_wiki_page() → LLM output
  │    │    └─ On failure → fall back to template
  │    ├─ Else → template-based rendering:
  │    │    ├─ _format_node() for each node
  │    │    └─ _linkify() post-pass
  │    └─ Write <slug>.md
  ├─ Write index.md (links to all topic pages)
  └─ Return {"pages": N, "total_nodes": M}
```

The **CLI flow** adds:
```
reasons build-wiki -o DIR --status IN
  → parses args
  → calls api.build_wiki(...)
  → prints "Wiki written to DIR (N pages, M nodes)"
```

The **LLM flow** (tested in `TestBuildWikiLlm`):
```
build_wiki(..., model="claude", parallel=2)
  → for each topic (up to `parallel` concurrently):
       generate_wiki_page(topic, node_ids, details, "claude", 300)
         → _format_beliefs_for_prompt() → prompt string
         → reasons.llm.invoke_model(prompt, model, max_tokens)
         → returns Markdown string
       → write to file
  → if generate_wiki_page raises → fall back to _format_node template
```

## 6. Invariants

The tests collectively enforce these rules:

1. **Every node appears in exactly one topic group.** Unmatched nodes go to `"Other"`, never dropped. (Tested in `TestAssignTopics`.)
2. **Page filenames are always valid slugs.** Special characters, spaces, and empty results are handled; the fallback is `"other"`. (`TestPageName`.)
3. **Cross-references only link to known pages.** If a referenced node has no page, it appears as plain text, never as a broken link. (`TestFormatNode.test_no_links_for_unknown_nodes`, `test_duplicate_of_no_page`.)
4. **The index page is always created**, even for an empty DB (with "No beliefs found" text). (`TestBuildWikiApi.test_empty_db`.)
5. **Slug collisions are resolved**, never silently overwriting. Two topics that slugify to the same name must produce two distinct files. (`test_duplicate_slug_collision`.)
6. **LLM failure never prevents wiki generation.** The fallback to template rendering must produce valid output. (`test_llm_failure_falls_back`.)
7. **`_linkify` processes longest IDs first** to prevent `node-alpha` from consuming `node-alpha-extended`. (`test_longest_first`.)
8. **Same-page references are not linkified** (no self-links). (`test_skips_same_page`.)

## 7. Error Handling

The tests encode specific error-handling contracts:

- **LLM failure → silent fallback.** `TestBuildWikiLlm.test_llm_failure_falls_back` patches `generate_wiki_page` to raise `RuntimeError("LLM failed")` and asserts the output page still contains the template-rendered `### node-a` header. The error is swallowed; no exception propagates to the caller.

- **Missing node in details dict → skipped gracefully.** `TestFormatBeliefsForPrompt.test_skips_missing_nodes` asserts that requesting a node not in `details` produces an empty string, not a `KeyError`.

- **CLI errors → non-zero exit code.** The `run_cli` helper captures `SystemExit` and returns the code. Tests assert `code == 0` for valid invocations. (Implicitly, invalid args would produce `code != 0`, though no test explicitly checks that here.)

- **Empty inputs → valid (degenerate) output.** `test_empty_db` and `test_empty_nodes` ensure that zero nodes/topics still produce a valid `index.md` and an empty dict respectively, rather than crashing.

- **`metadata: None` is handled.** `test_none_metadata` ensures the formatting code guards against `None` metadata without raising `AttributeError`.

---

## Topics to Explore

- [file] `reasons/build_wiki.py` — The implementation under test; understand the template rendering logic, the `_linkify` regex, and the parallel generation path (likely using `concurrent.futures`).
- [file] `reasons/llm.py` — The `invoke_model` abstraction that `generate_wiki_page` delegates to; understand how model selection, token limits, and provider routing work.
- [function] `reasons/api.py:build_wiki` — The public API entry point; see how it loads nodes from the DB, applies the status filter, and calls into `reasons.build_wiki`.
- [file] `tests/conftest.py` — Shared fixtures and any global mocking/patching that affects how `reasons.llm` or `reasons.storage` behave across the test suite.
- [general] `belief-graph-data-model` — Understanding the node structure (`text`, `truth_value`, `justifications`, `dependents`, `metadata`) is prerequisite to reading both the tests and the formatting code.

## Beliefs

- `build-wiki-falls-back-to-template-on-llm-error` — If `generate_wiki_page` raises any exception, `build_wiki` silently falls back to the `_format_node` template and still writes a valid `.md` file; the caller never sees the error.
- `_linkify-skips-same-page-references` — The `_linkify` post-processing pass never inserts a Markdown link for a node that resides on the same page as the current content, preventing redundant self-links.
- `_assign_topics-uses-prefix-matching` — Topic assignment is determined by checking whether the topic string is a prefix of the node ID; the first matching topic wins, and unmatched nodes are grouped under the literal key `"Other"`.
- `build-wiki-produces-index-even-for-empty-db` — `api.build_wiki` always writes an `index.md` file, even when the database contains zero nodes; in that case the index contains the text "No beliefs found".
- `_page-name-falls-back-to-other-on-empty-slug` — When `_page_name` slugifies a topic string and the result is empty (e.g., all special characters), it returns the literal string `"other"` rather than an empty filename.
