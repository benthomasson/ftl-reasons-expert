---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_build_wiki.py

# `tests/test_build_wiki.py`

## Purpose

This file is the pytest suite for the `build-wiki` feature. It defines the expected contracts for:

- helper functions inside `reasons.build_wiki`
- the core `build_wiki(...)` orchestration function
- the public `reasons.api.build_wiki(...)` wrapper
- the CLI command `reasons build-wiki`
- LLM-backed wiki generation and fallback behavior

It is not production code. It is a contract specification: if `reasons/build_wiki.py`, `reasons/api.py`, or `reasons/cli.py` changes behavior, these tests are expected to catch it.

The repository context also shows a parallel `reasons_lib` tree, but this test file imports from `reasons`, not `reasons_lib`.

---

## Key Components

### `run_cli(*args, db_path=None)`

A helper for exercising the real CLI entry point.

It:

- imports `reasons.cli.main`
- builds `sys.argv` as `["reasons", "--db", db_path, *args]` when `db_path` is provided
- patches `sys.argv`, `sys.stdout`, and `sys.stderr`
- calls `main()`
- catches `SystemExit` and converts `e.code` to an integer exit code
- returns `(stdout, stderr, code)`

This means CLI tests invoke the same path users would use, including global options before the subcommand.

Example test usage:

```python
stdout, stderr, code = run_cli("build-wiki", "-o", output_dir, db_path=db)
```

### `db` fixture

Creates a temporary SQLite database under pytest’s `tmp_path` and seeds it through the public API:

```python
api.add_node("caching-strategy-redis", ...)
api.add_node("caching-ttl-policy", ...)
api.add_node("testing-unit-coverage", ...)
api.add_node("testing-integration-api", ...)
api.add_node("deploy-ci-pipeline", ..., sl="testing-unit-coverage")
```

That last node links `deploy-ci-pipeline` to `testing-unit-coverage` via `sl`, which is later used to verify cross-reference rendering.

The fixture returns the database path, not a connection object.

### Test classes

#### `TestAssignTopics`

Tests `_assign_topics(node_ids, topics)`.

Contracts:

- node IDs matching a topic are grouped under that topic
- non-matching IDs go to `"Other"`
- empty node list returns `{}`
- empty topic list puts every node under `"Other"`

#### `TestPageName`

Tests `_page_name(topic)`.

Contracts:

- simple names pass through, e.g. `"caching"` → `"caching"`
- spaces and special characters are normalized to hyphens, e.g. `"My Topic!"` → `"my-topic"`
- invalid/empty result falls back to `"other"`

#### `TestFormatNode`

Tests `_format_node(node_id, detail, node_to_page, all_details=None)`.

This is the main Markdown renderer for one belief node.

Contracts:

- emits a heading like `### caching-redis`
- emits `**Status:** IN`
- includes the belief text
- renders antecedents as `**Depends on:**`
- renders dependents as `**Supports:**`
- links cross-references only when the target exists in `node_to_page`
- leaves unknown node IDs as plain text, not broken links
- handles `metadata: None` safely
- renders `duplicate_of` metadata as `**Duplicate of:** ...`
- renders `superseded_by` metadata as `**Superseded by:** ...`
- when `all_details` is supplied, inspects `outlist` defeater nodes and renders `**Defeated by:** ...`
- includes defeater metadata such as `defeater_type` and optional `defeat_reason_type`
- does not render a `Defeated by` line when `all_details` is omitted

Cross-reference link format is enforced as:

```markdown
[node-id](page.md#node-id)
```

#### `TestBuildWiki`

Tests `build_wiki(details, groups, output_dir, ...)` directly without a database.

Contracts:

- creates `index.md`
- creates one topic page per group
- returns stats containing at least `pages` and `total_nodes`
- index page links to topic pages, e.g. `[mytopic](mytopic.md)`
- index page includes wording like `1 beliefs across 1 pages`
- topic pages include `[Back to index](index.md)`
- a topic named `"index"` must not overwrite `index.md`; it is written as `index-topic.md`
- duplicate slug topics, such as `"My Topic"` and `"My-Topic!"`, must produce two distinct files

#### `TestBuildWikiApi`

Tests `api.build_wiki(output_dir=..., status=..., model=..., db_path=...)`.

Contracts:

- reads beliefs from the SQLite DB and writes a wiki directory
- with five seeded nodes, returns `total_nodes == 5`
- creates `index.md`
- respects a `status="IN"` filter after a node is retracted
- empty DB produces `total_nodes == 0`, `pages == 0`, and an index containing `"No beliefs found"`
- cross references from stored support links resolve into rendered Markdown

#### `TestBuildWikiCli`

Tests the user-facing CLI command.

Contracts:

- `build-wiki --help` exits successfully and mentions the command/wiki
- `build-wiki -o <dir>` creates `index.md`
- success prints `"Wiki written to"` to stdout
- CLI accepts `--status IN`

#### `TestLinkify`

Tests `_linkify(content, current_page, node_to_page, known_node_ids)`.

Contracts:

- turns known node IDs inside text into cross-page Markdown links
- does not link references that belong to the same page
- does not double-link content that is already a Markdown link
- replaces longest node IDs first, so `node-alpha-extended` is not partially treated as `node-alpha`

#### `TestFormatBeliefsForPrompt`

Tests `_format_beliefs_for_prompt(node_ids, details)`.

Contracts:

- emits a prompt-friendly text block for each node
- includes `### node-id`, `Status: IN`, `Text: ...`, `Depends on: ...`, `Supports: ...`
- silently skips missing node IDs
- returns empty/whitespace when nothing matches

#### `TestGenerateWikiPage`

Tests `generate_wiki_page(topic, node_ids, details, model, max_tokens)`.

Contracts:

- calls `reasons.llm.invoke_model(...)`
- returns the generated string
- builds a prompt containing the topic name and node IDs

The test patches:

```python
patch("reasons.llm.invoke_model", return_value="Generated wiki content")
```

so the production implementation must call `reasons.llm.invoke_model` through that module path.

#### `TestBuildWikiLlm`

Tests LLM integration in `build_wiki`.

Contracts:

- when `model` is supplied, `build_wiki` calls `generate_wiki_page`
- when `model` is not supplied, `generate_wiki_page` is not called
- if `generate_wiki_page` raises `RuntimeError`, `build_wiki` still writes a usable static page
- `api.build_wiki(model="claude")` passes the model through and calls `generate_wiki_page` once per generated page
- `build_wiki(..., model="claude", parallel=2)` still generates every page

The tests patch:

```python
patch("reasons.build_wiki.generate_wiki_page", ...)
```

so `build_wiki` must resolve/call `generate_wiki_page` at the module level in `reasons.build_wiki`.

---

## Patterns

- **Class-based pytest organization**: related tests live under `TestAssignTopics`, `TestBuildWiki`, etc.
- **Fixture-driven integration setup**: `db` fixture seeds a real temporary DB via the public API.
- **Unit tests for private helpers**: the suite imports private functions such as `_assign_topics`, `_page_name`, `_format_node`, and `_linkify`. This is intentionally tight coupling to internals.
- **Mocking at module boundaries**: LLM calls are mocked with `unittest.mock.patch` so tests do not depend on network/model behavior.
- **CLI capture pattern**: `run_cli` patches `sys.argv`, `sys.stdout`, `sys.stderr`, and catches `SystemExit`.
- **Filesystem assertions using `tmp_path`**: tests read generated Markdown files directly from the temporary output directory.
- **Fallback testing**: the LLM failure test ensures the wiki builder degrades to deterministic static Markdown when generation fails.

---

## Dependencies

### This file imports

- standard library: `os`, `sys`, `StringIO`, `unittest.mock.patch`
- pytest
- `reasons.api`
- private helpers and public functions from `reasons.build_wiki`
- `reasons.cli.main` inside `run_cli`

### It indirectly exercises

- `reasons.llm.invoke_model`
- `reasons.api.add_node`
- `reasons.api.retract_node`
- `reasons.api.build_wiki`
- CLI argument parsing and output formatting

### What depends on this file

No production code imports tests. This file is discovered and run by pytest.

Because it imports private helpers by name, refactoring `reasons.build_wiki` internals requires updating this test file even if public behavior remains unchanged.

---

## Flow

### Unit-level flow

Most helper tests follow:

1. construct plain dictionaries/lists
2. call a private helper directly
3. assert generated Markdown or grouped data

Example: `_format_node` tests build `detail`, `node_to_page`, and sometimes `all_details`, then assert exact Markdown fragments.

### API-level flow

API tests:

1. use the `db` fixture to create a temporary SQLite database
2. call `api.build_wiki(output_dir=..., status=..., model=..., db_path=...)`
3. inspect returned stats and generated Markdown files

The production `api.build_wiki` is responsible for loading DB state and delegating to `reasons.build_wiki.build_wiki`.

### CLI-level flow

CLI tests:

1. call `run_cli("build-wiki", ...)`
2. `run_cli` patches `sys.argv` to simulate process arguments
3. calls `reasons.cli.main()`
4. captures stdout/stderr and `SystemExit` code
5. asserts output directory and printed message

The CLI delegates to the API/build-wiki implementation; the test does not call the API directly in CLI tests.

### LLM flow

LLM-backed tests patch `generate_wiki_page` inside `reasons.build_wiki`:

- `build_wiki(..., model="claude")` calls patched `generate_wiki_page`
- the returned string is written into the generated topic page
- `parallel=2` still writes all expected topic pages
- without `model`, the patch is not called
- if the patch raises, static fallback Markdown is written

The lower-level `TestGenerateWikiPage` separately verifies that `generate_wiki_page` itself calls `reasons.llm.invoke_model`.

---

## Invariants

The tests enforce these rules:

- `index.md` is always written by `build_wiki`, even for empty input.
- Topic page filenames are slugified through `_page_name`.
- A topic named `index` must not overwrite the index page; it becomes `index-topic.md`.
- Different topics that slugify to the same value must still produce distinct files.
- Generated topic pages contain a backlink to `index.md`.
- Cross-reference links use `page.md#node-id` anchors.
- Unknown node references remain plain text and are not Markdown-linked.
- Duplicate and superseded metadata links are only linked when the target page is known.
- `metadata: None` must not crash formatting.
- Defeater display requires `all_details`; without it, `_format_node` does not emit a `Defeated by` line.
- `_assign_topics([])` returns `{}`, not a dict with an empty `"Other"` group.
- `_format_beliefs_for_prompt` skips missing nodes and returns empty/whitespace for an empty match set.
- `_linkify` does not re-link already-linked Markdown, and it processes longest node IDs first.
- `api.build_wiki(status="IN")` filters out non-IN beliefs.
- Empty DB returns `total_nodes == 0` and `pages == 0`.

---

## Error Handling

Notable behavior:

- `run_cli` catches `SystemExit` only. Other exceptions propagate and would fail the test.
- `build_wiki` is expected to catch failures from LLM generation. The test injects `RuntimeError("LLM failed")` and expects a valid static page instead of a crash.
- Missing node references are tolerated:
  - `_format_node` leaves unknown node IDs unlinked
  - `_format_beliefs_for_prompt` skips missing nodes
  - missing `node_to_page` entries cause plain text output instead of broken links
- `metadata=None` is tolerated by `_format_node`.
- Empty DB is not an error; it produces a valid `index.md` containing `"No beliefs found"`.
- `_page_name("!!!")` does not return an invalid empty filename; it falls back to `"other"`.

---

## Topics to Explore

- [file] `reasons/build_wiki.py` — Production implementation under test: topic assignment, page naming, Markdown rendering, LLM integration, and collision handling.
- [function] `reasons/build_wiki.py:build_wiki` — Core orchestration function responsible for index generation, page writing, parallel mode, and LLM fallback.
- [function] `reasons/build_wiki.py:_format_node` — Most complex renderer; handles cross-references, duplicate/superseded metadata, and defeater display.
- [file] `reasons/api.py` — Public `api.build_wiki` wrapper that loads DB state, applies status filtering, and delegates to `build_wiki`.
- [file] `reasons/cli.py` — CLI parsing and `build-wiki` command wiring, including output messages and `--status` handling.

## Beliefs

- `db-fixture-seeds-five-beliefs-through-api` — The `db` fixture creates five nodes in a temporary SQLite database via `reasons.api.add_node`, including a support link from `deploy-ci-pipeline` to `testing-unit-coverage`.
- `format-node-linkifies-known-references-only` — `_format_node` renders cross-references as `[id](page.md#id)` only when the target ID exists in `node_to_page`; otherwise it leaves the ID as plain text.
- `build-wiki-falls-back-on-llm-errors` — When `model` is supplied, `build_wiki` uses `generate_wiki_page`, but a `RuntimeError` from generation does not prevent static `_format_node` Markdown from being written.
- `build-wiki-always-writes-index-md` — `build_wiki` creates `index.md` even when there are no beliefs, and that page contains `"No beliefs found"` for an empty database.
- `linkify-replaces-longest-node-ids-first` — `_linkify` processes node IDs longest-first so `node-alpha-extended` is linked as a single token rather than partially linking `node-alpha`.
