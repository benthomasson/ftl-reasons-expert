---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_ask.py

# `tests/test_ask.py`

## Purpose

This file is the pytest specification for the `reasons.ask` subsystem. It verifies three broad layers:

1. **Pure helpers** — tool-call parsing, prompt construction, metadata stripping, and source-chunk search.
2. **`ask()` orchestration** — FTS5 belief search, optional source-document search, LLM tool-call loops, simple one-shot answering, natural-language mode, dual-answer merge mode, no-synthesis fallback, and failure fallbacks.
3. **CLI integration** — `reasons ask --no-synth` behavior through `reasons.cli.main`.

It does not implement production behavior. Instead, it encodes the expected contracts of `reasons.ask` and uses mocked LLM calls so tests are deterministic.

---

## Key Components

### `db_path` fixture

```python
@pytest.fixture
def db_path(tmp_path):
    return str(tmp_path / "test.db")
```

Creates an isolated SQLite database path per test.

### `run_cli(*args, db_path=None)`

An in-process CLI harness. It:

- Builds `argv = ["reasons", "--db", db_path, ...]`
- Patches `sys.argv`, `sys.stdout`, and `sys.stderr`
- Calls `reasons.cli.main()`
- Catches `SystemExit` and returns `(stdout, stderr, exit_code)`
- Returns exit code `0` if `main()` returns normally

This lets tests initialize databases and add beliefs via the real CLI without spawning a subprocess.

---

### `TestExtractToolCall`

Tests `extract_tool_call(text)`.

Contract encoded by tests:

- Scans text for a JSON object with a `"tool"` key.
- Ignores plain text and malformed JSON.
- Returns the **first** valid tool-call object.
- Returns `None` if no tool call exists.
- Empty input returns `None`.

Examples:

```python
'{"tool": "search_beliefs", "query": "propagation"}'
# -> {"tool": "search_beliefs", "query": "propagation"}
```

Malformed JSON before a valid tool call is skipped.

---

### Prompt builders

#### `build_ask_prompt(question, context, tool_history=None)`

Builds the full interactive prompt used when the LLM is allowed to request search tool calls.

Tests assert:

- Contains the user question and belief context.
- Contains the `search_beliefs` tool definition and `"tool"`.
- Omits `"Additional search results"` when `tool_history` is empty.
- Includes `"Additional search results"` and prior tool labels/results when history exists.

#### `build_final_prompt(question, context)`

Builds a final synthesis prompt without tool definitions.

Tests assert:

- Contains question and context.
- Does not contain `"search_beliefs"` or `"tool"`.

#### `build_simple_prompt(question, context, natural=False)`

Builds a one-shot prompt with no tool access.

Tests assert:

- Contains question and context.
- No `search_beliefs` and no `"tool"`.
- Default mode includes `"Cite belief IDs in [brackets]"`.
- `natural=True` removes the citation instruction and adds `"plain natural language"`.

---

### `ask(...)`

The central orchestration function under test. Tests call it with combinations of:

- `db_path`
- `no_synth=True`
- `format="markdown"`
- `simple=True`
- `natural=True`
- `sources_db=...`
- `dual=True`

Key expected behaviors:

#### No-synthesis mode

`ask(..., no_synth=True)` bypasses the LLM and returns compact FTS5 search output.

- Default compact output contains `[IN] <belief-id>` lines and an em dash separator.
- Markdown format includes markdown-like output such as `##`, `**`, or belief IDs.
- No matches returns text containing `"No results"`.

#### Full/default mode

The default mode supports an iterative LLM tool-call loop:

1. Search beliefs and optionally source documents.
2. Ask the LLM with `build_ask_prompt`.
3. If the LLM returns a `search_beliefs` tool call, run the search and retry with tool history.
4. After at most three tool-call rounds, call final synthesis with `build_final_prompt`.
5. If final synthesis times out, errors, or returns another tool call, fall back to accumulated belief/search context.

Tests verify call counts:

- Tool call then answer: 2 LLM calls.
- Three tool calls then final answer: 4 LLM calls.
- Extra synthesis timeout or tool-call response: returns belief context, not raw tool JSON.

#### Simple mode

`ask(..., simple=True)` makes a single LLM call using `build_simple_prompt`, with no tool loop and no tool definition.

If there are no matching beliefs and no matching source documents, the tests expect `NO_BELIEFS_MSG` without requiring an LLM response.

#### Natural mode

`natural=True`:

- Applies `_strip_belief_metadata` before prompting.
- Removes markdown belief headers and metadata such as `**Status:**`, `**Source:**`, `###`, and related-node sections.
- Removes citation instructions.
- Adds `"plain natural language"`.
- Propagates natural behavior into tool-call results and dual-mode TMS prompt.

#### Source documents

When `sources_db` is provided, source-document search results are appended as `"Source Documents"` context.

Tests verify:

- Source text and filenames appear in the LLM prompt.
- Source documents survive a tool-call retry.
- Sources alone can produce an answer when beliefs are empty.
- No beliefs and no source matches returns `NO_BELIEFS_MSG`.

#### Dual mode

`dual=True` requires `sources_db`. It performs three LLM calls:

1. TMS/belief answer.
2. FTS/source answer.
3. Merge answer.

Without `sources_db`, it raises:

```python
ValueError("--dual requires --full-sources")
```

---

### `_invoke_claude(prompt)`

Tests verify that `_invoke_claude` raises `FileNotFoundError` mentioning `"claude"` when the Claude executable is unavailable.

The test patches:

```python
reasons.llm.shutil.which
```

This is notable: it couples `_invoke_claude` to `reasons.llm`’s executable lookup behavior.

---

### `_strip_belief_metadata(context)`

Removes markdown formatting and metadata from belief context before natural-language prompting.

Tests cover removal of:

- `### belief-id`
- `**Status:**`
- `**Source:**`
- `**Depends on:**`
- `**Justification:**`
- `**Supported by:**`
- `**Supports:**`
- `**Depended on by:**`
- `**Related nodes:**` and following list entries
- Horizontal rule `---`

It also:

- Preserves plain text.
- Collapses excessive blank lines.
- Handles `""` and `None`; `_strip_belief_metadata(None) is None`.

---

### `_search_source_chunks(query, sources_db, top_k=...)`

Searches the source-document FTS5 table.

Tests build a minimal schema:

```sql
CREATE TABLE chunks (
    id INTEGER PRIMARY KEY,
    text TEXT,
    cluster TEXT,
    filename TEXT,
    section TEXT
);

CREATE VIRTUAL TABLE chunks_fts USING fts5(
    text,
    content=chunks,
    content_rowid=id
);

INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild');
```

Expected behavior:

- Returns markdown-formatted matching chunks.
- Includes filename in headers.
- Includes section as a `> Section` line when present.
- Omits section separator when `section` is `NULL`.
- No matches returns `""`.
- Single-character words are filtered.
- Common stop words are filtered.
- Bad database path returns `""`.
- `top_k` limits result count, e.g. `top_k=1` produces `### [1]` but not `### [2]`.

---

### `NO_BELIEFS_MSG`

A constant returned when `ask()` has no usable belief/source context and cannot synthesize an answer, or when the LLM fails before useful context is gathered.

Tests frequently assert exact equality:

```python
assert result == NO_BELIEFS_MSG
```

---

## Patterns

### Class-per-behavior grouping

Each test class targets a specific function or behavior:

- `TestExtractToolCall`
- `TestBuildAskPrompt`
- `TestAskNoSynth`
- `TestAskWithMockedLLM`
- `TestAskSimple`
- `TestAskNatural`
- `TestAskWithSources`
- `TestAskDual`

This keeps contracts discoverable.

### In-process CLI testing

`run_cli` avoids subprocess overhead and captures printed output by patching `sys.argv`, `sys.stdout`, and `sys.stderr`.

### Mocked LLM boundary

Most LLM behavior is patched at:

```python
patch("reasons.ask.invoke_model", ...)
```

This isolates `ask()` orchestration from real model calls.

### Stateful `side_effect` mocks

Multi-turn behavior is simulated with call counters:

```python
calls = [0]

def mock_invoke(prompt, model="claude", timeout=300):
    calls[0] += 1
    if calls[0] == 1:
        return '{"tool": "search_beliefs", "query": "retraction"}'
    return "Answer..."
```

This verifies call counts and prompt content across iterations.

### Minimal SQLite FTS5 fixtures

Several test classes define their own `sources_db` fixture. Each creates just enough of the production schema to test `_search_source_chunks` and source integration.

---

## Dependencies

### Direct imports

- `sqlite3` — builds source DB fixtures.
- `subprocess` — simulates `TimeoutExpired`.
- `sys` — patched for CLI invocation.
- `io.StringIO` — captures CLI stdout/stderr.
- `unittest.mock.patch` — mocks LLM, executable lookup, CLI streams.
- `pytest` — fixtures, `raises`.
- `reasons.ask` — production functions/constants under test.
- `reasons.cli.main` — CLI entry point used to initialize DBs and test `ask` command.

### Production modules indirectly coupled

- `reasons.ask.invoke_model` — patched in most LLM tests.
- `reasons.llm.shutil.which` — patched to simulate missing Claude executable.
- SQLite FTS5 — required by source-document tests.

### Imported by

No production module imports this test file. It is collected by pytest only.

---

## Flow

Typical test flow:

1. Obtain isolated SQLite path from `db_path`.
2. Call `run_cli("init", db_path=...)`.
3. Add beliefs with `run_cli("add", id, text, db_path=...)`.
4. Call either:
   - `ask(...)` directly, usually with `reasons.ask.invoke_model` patched.
   - `run_cli("ask", ..., "--no-synth", ...)` for CLI integration.

For a full-mode `ask()` call, the expected orchestration is:

```text
query
  -> initial belief search
  -> optional source-chunk search
  -> build ask prompt
  -> invoke_model
       -> if search_beliefs tool call:
            run belief search
            append results
            rebuild prompt with tool history
            invoke_model again
       -> repeat up to bounded tool rounds
  -> build final prompt / simple prompt
  -> invoke_model for final synthesis
  -> return answer or accumulated belief context on failure
```

For `simple=True`:

```text
query
  -> initial search
  -> if no beliefs and no source matches: return NO_BELIEFS_MSG
  -> build simple prompt
  -> single invoke_model call
  -> return answer or fallback belief context
```

For `dual=True`:

```text
query
  -> TMS/belief answer
  -> FTS/source answer
  -> merge prompt
  -> merged answer
```

For `no_synth=True`:

```text
query
  -> FTS search
  -> format compact or markdown
  -> return text
```

---

## Invariants

- `extract_tool_call` returns the first valid JSON object with a `tool` key and ignores malformed JSON lines.
- The full-mode LLM tool loop is bounded: tests expect at most three `search_beliefs` tool rounds before final synthesis.
- Final synthesis must not leak raw `search_beliefs` JSON back to the caller.
- `dual=True` requires `sources_db`; otherwise `ValueError("--dual requires --full-sources")` is raised.
- `natural=True` removes belief metadata and citation instructions from prompts.
- Non-natural prompts include the citation instruction.
- `simple=True` prompts contain no tool definition.
- Source documents must be preserved across tool-call retries.
- `_search_source_chunks` never raises on a bad database path; it returns `""`.
- `_strip_belief_metadata(None)` returns `None`.
- `NO_BELIEFS_MSG` is the exact fallback when no usable context exists and the LLM cannot produce an answer.

---

## Error Handling

### `_invoke_claude`

If the Claude executable is missing, `_invoke_claude` raises `FileNotFoundError` containing `"claude"`.

### `ask()` LLM failures

`ask()` catches:

- `subprocess.TimeoutExpired`
- `RuntimeError`

If belief/source context has already been gathered, it returns that context instead of raising.

If no useful context exists, it returns `NO_BELIEFS_MSG`.

Examples encoded by tests:

- Timeout on empty network → `NO_BELIEFS_MSG`
- Runtime error on empty network → `NO_BELIEFS_MSG`
- Timeout after successful tool search → return accumulated beliefs
- Timeout during extra/final synthesis → return accumulated beliefs, not raw tool JSON

### Unknown tools

If the LLM returns an unknown tool call, `ask()` returns the LLM response as-is rather than crashing.

### Source DB errors

`_search_source_chunks` swallows bad-database errors and returns an empty string.

### CLI exits

`run_cli` catches `SystemExit` and returns the exit code, so CLI tests do not abort the pytest process.

---

## Topics to Explore

- [file] `reasons/ask.py` — The production implementation under test; inspect the actual tool-loop bounds and fallback paths.
- [function] `reasons/ask.py:ask` — Central orchestration: full, simple, natural, no-synth, source, and dual modes.
- [function] `reasons/ask.py:_search_source_chunks` — FTS5 source-chunk query construction, stop-word filtering, and markdown formatting.
- [function] `reasons/cli.py:main` — How CLI arguments such as `--no-synth`, `--format`, and `--db` map into `ask()`.
- [general] `dual-answer-merge-strategy` — How TMS and FTS answers are combined and why a third LLM call is used.

---

## Beliefs

- `ask-no-synth-bypasses-llm` — When `ask(..., no_synth=True)` is called, the tests expect formatted FTS5 search results rather than an LLM synthesis.
- `dual-requires-sources-db` — `ask(..., dual=True)` without `sources_db` raises `ValueError` with `--dual requires --full-sources`.
- `tool-call-loop-is-bounded` — Full mode permits at most three `search_beliefs` tool rounds before a final synthesis call, as verified by call-count tests.
- `final-synthesis-falls-back-to-beliefs` — If final synthesis times out or returns another tool call, `ask()` returns accumulated belief context rather than raw `search_beliefs` JSON.
- `natural-mode-strips-belief-metadata` — When `natural=True`, prompts omit `**Status:**`, `###` headers, and citation instructions, and include `plain natural language`.
