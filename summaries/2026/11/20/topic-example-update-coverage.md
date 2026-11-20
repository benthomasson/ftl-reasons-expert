---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The docstring mentions update support, but this file contains no update tests; inspect whether update actually supports examples.

## Explanation

`tests/test_example.py:1` says:

> Tests for the `--example` flag on add, update, and show commands.

But the actual test file does not contain update tests. The 110-line file defines only:

- `TestAddNodeExample` — API `add_node(... example=...)`
- `TestCmdAddExample` — CLI `add --example`
- `TestCmdShowExample` — CLI `show` rendering `example`

There is no `TestUpdate...` class, no `test_update_*` function, and no call to an update API or CLI command in the observed test file.

The implementation evidence also points to add-only example support:

- `reasons/api.py:add_node` accepts `example: str | None = None` at line 289, forwards it for PostgreSQL at line 319, and stores it with `metadata["example"] = example` at lines 379–380.
- `reasons/pg.py` similarly stores `metadata["example"]` for the add path at lines 222–223.
- `reasons/cli.py:3085` defines an `--example` argument, and `reasons/cli.py:59` passes `example=getattr(args, "example", None)`, but the observed snippets do not show this being wired to an `update` command.
- The gathered searches found zero definitions for symbol `update` and zero matches for `def update`.
- `reasons/propose_update.py:format_proposal_markdown` renders a textual command like `reasons update <id> "..."`, but that is proposal formatting, not proof that `update` accepts an `example`.

So the docstring is stale or incorrect with respect to update tests. On the observed evidence, update does **not** demonstrably support examples; the only implemented and tested `example` plumbing is for `add` and `show`. If an update command exists outside the captured observations, its example handling is not shown here and needs direct inspection.

## Topics to Explore

- [file] `tests/test_example.py` — Confirms that the file covers only add/show example behavior despite the docstring mentioning update.
- [function] `reasons/api.py:add_node` — Shows the canonical example-handling path that an update implementation would need to mirror.
- [file] `reasons/cli.py` — Check argparse wiring to determine whether `--example` is attached only to `add` or also to `update`.
- [general] `update-command-implementation` — Search for the actual CLI/API update code to verify whether any update path accepts and persists `example`.
- [file] `reasons/propose_update.py` — Clarifies that this module formats update proposals and emits `reasons update` commands; it is not the update implementation.

## Beliefs

- `docstring-lists-update-tests-but-file-lacks-them` — `tests/test_example.py:1` claims update coverage, while the file defines only add and show test classes.
- `add-node-stores-example-in-metadata` — `reasons/api.py:add_node` sets `metadata["example"]` when `example` is not `None` at lines 379–380.
- `pg-add-path-stores-example-in-metadata` — `reasons/pg.py` sets `metadata["example"]` when `example` is not `None` at lines 222–223.
- `update-symbol-search-returned-zero` — The gathered searches found zero definitions for symbol `update` and zero matches for `def update`.
- `cli-example-flag-wiring-is-not-shown-for-update` — `reasons/cli.py:3085` defines `--example`, and line 59 passes it via `getattr`, but observed snippets do not tie it to an update command.
