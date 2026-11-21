---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: cmd_search_sources in reasons/cli.py

# `cmd_search_sources` — CLI handler for `reasons search-sources`

## Purpose

A thin CLI wrapper around `reasons.ask.search_source_chunks` that performs lexicon-style retrieval against an FTS5 chunks index (typically produced by a RAG ingestion pipeline) and prints the matching source chunks. It is the retrieval-only counterpart to `cmd_ask`: no LLM is invoked, no belief graph is read, and the belief DB (`reasons.db`) is not touched. It exists so that users can inspect "what raw source text matches this query?" independently of belief synthesis.

## Contract

**Preconditions**
- `args.query` is a non-empty string.
- `args.db` resolves to a path of a valid SQLite FTS5 database (the chunks index, e.g. `rag_fts.db`). This is a *different* `--db` than the global reasons DB — see the note on argument shadowing below.
- `args.top_k` is a positive integer (the parser defaults to 10; negative/zero values are not validated here and are passed through).
- `args.format` is `"text"` or `"json"` (parser enforces via `choices`).
- The FTS schema returns rows with at least a `filename` and a `text` column; `section` is optional.

**Postconditions**
- On success, chunk text has been written to stdout in either `text` (headed by `### [n] filename[ > section]` blocks separated by `---`) or `json` (`json.dumps(results, indent=2)`) form.
- Always returns `None` — the function never returns data; all output is via `print`.

**Invariants / hidden assumptions**
- The function relies on `search_source_chunks` returning a list of JSON-serialisable dicts. A non-serialisable value (e.g. `bytes`) would make `json.dumps` raise.
- Each result row is assumed to contain a `filename` and `text` key. `row.get("section")` is used for the optional field, but `row["filename"]` and `row["text"]` are direct subscripts — a missing key raises `KeyError`.
- The empty-results path short-circuits before the format check, so `--format json` on zero hits prints the plain string `"No matching chunks found."`, not `[]`. Tooling that `jq`s the output will fail on empty results.

## Parameters

`args` is the argparse `Namespace` for the `search-sources` subparser, produced in `main()`:

| Attribute | Source | Meaning / notes |
|-----------|--------|-----------------|
| `query`     | positional                 | FTS5 search string. Piped `|` allows OR terms (per the parent `search` help, this is standard FTS5 syntax passed through). Not validated here. |
| `db`        | `--db` (subparser, **required**) | Path to the FTS5 chunks DB. Despite the same flag name as the top-level `--db`, argparse gives the *subparser's* `--db` precedence for this command, so `args.db` is the FTS DB. |
| `top_k`     | `--top-k`, default 10      | Passed straight into `search_source_chunks`. No upper bound; no rejection of 0 or negative values. |
| `format`    | `--format`, default `"text"` | Read via `getattr(args, "format", "text")` (defensive even though the parser always sets it). Only exact `"json"` triggers JSON; anything else falls to text. |

## Return Value

Always `None`. The caller (`commands[args.command](args)` in `main()`) discards it. All persistence/display is expressed as `print` side effects.

## Algorithm

1. **Lazy import** — `from .ask import search_source_chunks` is inside the function. `.ask` pulls in LLM clients and possibly heavy ML deps, so importing it at module load would slow every `reasons` invocation. Deferring it confines that cost to `search-sources` and `ask`.
2. **Retrieve** — `search_source_chunks(args.query, args.db, top_k=args.top_k)`. `db` here is the FTS5 index path, not the reasons DB.
3. **Empty guard** — If the returned list is empty, print `"No matching chunks found."` and return. This is a *user-facing* message, not a JSON payload, regardless of `--format`.
4. **JSON branch** — If `args.format == "json"`, emit `json.dumps(results, indent=2)` and return. Uses the indenting form for readability; arrays of objects preserve insertion order of results.
5. **Text branch** — For each row (1-indexed `enumerate`):
   - Build header `f"[{i}] {row['filename']}"`; append `f" > {row['section']}"` *only* if `row.get("section")` is truthy (empty-string and `None` are suppressed).
   - Print the header as a markdown level-3 heading: `print(f"### {header}\n")` → produces `### [1] file.py > Sec\n\n` (two newlines: one from the f-string, one from `print`).
   - Print `row["text"]` verbatim.
   - If not the last row, print `"\n---\n"` as a markdown horizontal rule separator.
   - The last row has no trailing separator, so the overall output ends with a single `\n`.

## Side Effects

- **Reads** the FTS5 SQLite database (I/O, plus any underlying file locks) via `search_source_chunks`.
- **Writes** to stdout (`print`).
- **Imports** `reasons.ask` lazily. After first invocation, `sys.modules` caches it — subsequent calls do not re-import but do re-enter the module already loaded.
- **No writes** to the reasons belief DB, no environment mutation, no state persisted between invocations.

## Error Handling

The function has **no `try`/`except`** and does *not* follow the `sys.exit(1)` pattern used by other CLI handlers. Consequence:

- If the FTS5 DB does not exist / is not a valid SQLite file / lacks the expected schema, the underlying `sqlite3` (or `os`) exception propagates out of `search_source_chunks` and the CLI dumps a Python traceback. For a user-facing tool this is arguably a bug — `reasons search-sources --db /nonexistent.db q` fails ungracefully — but it is the current behaviour and may be relied on by tests that use `pytest.raises`.
- `KeyError` from `row["filename"]` or `row["text"]` propagates if `search_source_chunks` returns rows missing those keys. This is a hidden contract.
- `json.dumps` will raise `TypeError` if any returned row contains non-JSON-serialisable values.
- The empty-result path is the only "soft" failure — nothing is raised.
- Notably, `--pg` is silently ignored. Unlike `cmd_ask`, this handler does not call `_require_sqlite`, because the FTS DB is independent of the belief DB and `--pg` is irrelevant to it.

## Usage Patterns

Typical invocation:

```
reasons search-sources "embedding model" --db rag_fts.db --top-k 5 --format json | jq .
```

or via code/tests:

```python
from reasons.cli import cmd_search_sources
cmd_search_sources(argparse.Namespace(
    query="embedding model", db="rag_fts.db", top_k=5, format="text"))
```

Caller obligations:
- Provide an `args` object whose attributes the function expects to *exist*. `query`, `db`, `top_k`, and `format` are all accessed; only `format` has a `getattr` fallback. Passing a partial namespace without `db`, `query`, or `top_k` raises `AttributeError`.
- Use the subparser (not just the outer parser) so that `args.db` points at the FTS DB, not the reasons DB.

**Argument-shadowing caveat (worth flagging to colleagues):** the top-level parser also defines `--db` with default `api.DEFAULT_DB`. The `search-sources` subparser re-defines `--db` as required. For this subcommand the subparser's definition wins, so `args.db` is the FTS DB, and `--db` *must* be supplied after the subcommand name — a `reasons --db=reasons.db search-sources q` invocation fails parsing because the subparser's required `--db` is missing.

## Dependencies

- `json` (module-level import in `cli.py`) — used for the JSON branch.
- `.ask.search_source_chunks` — the only external-behaviour dependency; imported lazily.
- `argparse` `Namespace` produced by the `search-sources` subparser in `main()` — the function's schema is effectively defined by the parser block around "search-sources", not by type annotations.
- Indirect: `sqlite3` / the SQLite FTS5 extension (via `search_source_chunks`), which the handler never touches itself but on which the whole command depends.

## Topics to Explore

- [function] `reasons/ask.py:search_source_chunks` — Implements the FTS5 query and shapes each row as `{filename, section?, text}`; the actual correctness of `search-sources` depends entirely on this function.
- [file] `reasons/ask.py` — Houses the RAG retrieval + LLM synthesis pipeline that `cmd_search_sources` deliberately bypasses; also shows what other return fields are available for future formatting.
- [file] `tests/test_search_sources.py` — Likely covers empty-result messaging, JSON serialisation, and section-in-header logic; use it as the executable spec for the format output.
- [file] `tests/test_cli.py` — Verifies end-to-end argument wiring, including that the subparser's `--db` is required and distinct from the global `--db`.
- [general] `argparse-subparser-db-shadowing` — The root cause of the "which `--db` is which?" confusion; worth understanding before adding new subcommands that also want their own DB argument.

## Beliefs

- `search-sources-db-shadows-global-db` — For the `search-sources` subcommand, `args.db` refers to the FTS5 chunks DB and overrides the top-level `--db` (reasons DB) default.
- `search-sources-db-is-required` — The subparser marks `--db` as `required=True`, so `reasons search-sources QUERY` fails without `--db FTS_DB`.
- `cmd-search-sources-returns-none` — `cmd_search_sources` always returns `None`; all output is emitted through `print` to stdout.
- `cmd-search-sources-lazy-imports-ask` — `search_source_chunks` is imported inside the function body to avoid loading `reasons.ask` for unrelated CLI commands.
- `cmd-search-sources-empty-result-skips-json` — On zero matches, `cmd_search_sources` prints `"No matching chunks found."` regardless of `--format`, so JSON consumers do not receive `[]` on empty results.
