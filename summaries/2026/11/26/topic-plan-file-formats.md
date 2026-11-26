---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The general design pattern of human-editable markdown ceremonies in this codebase (`parse_dedup_plan` / `write_dedup_plan` follow the same shape).

# The `write_*_plan` / `parse_*_plan` Ceremony Pattern

## The Core Idea: A Three-Phase Round-Trip With a Human in the Middle

The codebase implements what is effectively a **"propose → review → accept"** workflow, split across three functions that share a data shape:

| Phase | Function | Direction | Signature |
|---|---|---|---|
| Propose | `write_dedup_plan` (`reasons/api.py:5079`) | in-memory → markdown | `(clusters: list[dict], output_path: str) -> str` |
| Review | *(the human opens the file in an editor)* | external | — |
| Parse | `parse_dedup_plan` (`reasons/api.py:5107`) | markdown → in-memory | `(plan_text: str) -> list[dict]` |
| Apply | `apply_dedup_plan` (`reasons/api.py:5139`) | in-memory → DB mutation | `(plan: list[dict], db_path=...)` |

The `write_dedup_plan` docstring (captured at `reasons/api.py:5082`) makes the contract explicit in prose:

> "Format is parseable by `parse_dedup_plan()`. Each cluster lists the…"

And the `apply_dedup_plan` docstring (referenced at `reasons/api.py:5146`) tells us the shape of the parsed payload:

> `plan: list of {"keep": str, "retract": list[str]} from parse_dedup_plan`

So the data contract is a **list of dicts** with a canonical "keep something, retract a set" shape. The writer is responsible for emitting markdown such that this payload can be reconstructed; the parser is responsible for reconstructing it. **The two functions form a round-trip pair** — neither stands alone.

## Why This Shape (and Not Just Apply Directly)

Notes in `.claude/skills/reasons/SKILL.md:268` and `CLAUDE.md:93` are decisive here:

> "Reasons uses SQLite (`reasons.db`), not markdown. This provides ACID transactions during propagation cascades…"

So the markdown file is **not** the source of truth. It is a **review surface** — a serialization of a *proposed* mutation that a human can inspect and edit before the mutation is committed. This is why the pattern is split across three functions instead of one `deduplicate --auto` command. The compute step (`write_dedup_plan`) is cheap and non-destructive; the destructive step (`apply_dedup_plan`) is deliberately gated behind a file the user has touched.

## The CLI Proves the Ceremony Is Intentional

The CLI wiring in `reasons/cli.py` makes the split explicit and non-simultaneous:

- **Line 1480** — `api.write_dedup_plan(result["clusters"], output)` writes the plan file from the `deduplicate` command. No application happens here.
- **Lines 1437–1441** — a *separate* command reads the file, calls `api.parse_dedup_plan(accept_path.read_text())`, and only then calls `api.apply_dedup_plan(plan, db_path=args.db)`.

The gap between these two invocations is exactly where the human edits. The parser (`parse_dedup_plan`) takes *text* — not a Python object — which is the design signal that the input may have been modified by something outside the process (a text editor).

## The Same Shape Repeats Across Ceremonies

The `find_related_plan_parsers_writers` observation confirms the design generalizes. `contradictions` follow the identical trio:

- `write_contradiction_plan` — `reasons/api.py:4836`
- `parse_contradiction_plan` — `reasons/api.py:4867`

And in `CHANGELOG.md:70` the workflow is named as such:

> "`deduplicate` command with Jaccard similarity clustering, auto-retract, and **review-then-accept workflow**"

This tells us the pattern is the **canonical ceremony for human-in-the-loop batch mutations** in this codebase — not a one-off. Any new "show me a batch of proposed changes, let me edit them, then apply" feature should follow the same shape:

1. `write_<thing>_plan(results, path) -> str`
2. `parse_<thing>_plan(text) -> list[dict]`
3. `apply_<thing>_plan(plan, db_path=...)`

## Tests Encode the Contract

The tests in `tests/test_derive.py` are the executable specification of the design pattern:

- **`test_write_and_parse_dedup_plan`** (`tests/test_derive.py:801`) — asserts round-trip fidelity: `write_dedup_plan` then `parse_dedup_plan` yields the original data.
- **`test_dedup_plan_end_to_end`** (`tests/test_derive.py:823`) — exercises all three phases (`write` → `parse` → `apply`) together.
- **`test_dedup_plan_user_can_edit_kept`** (`tests/test_derive.py:882`) — this is the most telling test. It verifies that a user can *change which node is kept* between `write` and `parse`, and that `apply` honors the edit. In other words, the design pattern's central guarantee — human editability — is a testable invariant, not just a documentation claim.

Together these three tests are effectively a template any future ceremony function should pass: round-trip fidelity, end-to-end application, and post-write editability.

## Design Invariants Worth Internalizing

Pulling the observations together, the pattern's invariants are:

1. **Round-trip symmetry** — writer is the parser's evil twin; they must agree on format.
2. **Text is the interchange type** — the parser accepts `str`, not a structured object, proving the file may have been edited externally.
3. **Compute and apply are separated by a file boundary** — the writer never mutates the DB; the applier never computes. This is what makes the file reviewable.
4. **The DB remains the single source of truth** — the markdown file is a transient proposal, not a persistent belief store (`CLAUDE.md:93`, `SKILL.md:268`).
5. **The same trio shape generalizes** — dedup and contradiction ceremonies both follow it, so new ceremonies should too.

## What the Observations Do Not Establish

The observations do **not** include the bodies of `write_dedup_plan` or `parse_dedup_plan`, so the concrete markdown syntax (heading depth, bullet format, how `keep:` vs `retract:` are spelled) cannot be described here. They also don't establish how tolerant the parser is to freeform edits (e.g., comment lines, whitespace changes, reordered fields) — only that a `keep` value can be changed (`tests/test_derive.py:882`). If you need to add a new ceremony, reading the actual function bodies in `reasons/api.py:5079-5140` is worth doing before designing one.

## Topics to Explore

- [function] `reasons/api.py:write_dedup_plan` — The writer's exact markdown syntax, which defines the parser's grammar
- [function] `reasons/api.py:parse_dedup_plan` — How tolerant the parser is of user edits (whitespace, unknowns, reordering)
- [function] `reasons/api.py:apply_dedup_plan` — What DB mutation each parsed `{keep, retract}` pair triggers and how cascades fire
- [file] `reasons/cli.py` — The `deduplicate` and `accept` subcommands (around lines 1437 and 1480) that wire the ceremony to the shell
- [general] `contradiction-plan-ceremony` — Confirm the contradiction trio (`write`/`parse`/`apply`) is truly structurally identical to dedup, or has its own quirks

## Beliefs

- `write-dedup-plan-produces-parseable-markdown` — The writer's output is contractually parseable by the parser, enforced by `test_write_and_parse_dedup_plan` (`tests/test_derive.py:801`) and stated in the docstring at `reasons/api.py:5082`.
- `dedup-plan-requires-parse-before-apply` — `apply_dedup_plan` consumes the output of `parse_dedup_plan`, never raw markdown; the CLI (`reasons/cli.py:1437-1441`) sequences parse before apply.
- `human-editable-markdown-is-review-surface-not-source-of-truth` — The SQLite DB is authoritative; the plan markdown is a transient proposal surface (`CLAUDE.md:93`, `.claude/skills/reasons/SKILL.md:268`).
- `plan-ceremony-requires-explicit-accept-step` — `write_dedup_plan` does not apply its own output; a separate CLI invocation must parse and apply the file (`reasons/cli.py:1437-1441` vs `:1480`).
- `contradiction-and-dedup-plans-share-shape` — Both ceremonies expose the same `write_*_plan` / `parse_*_plan` / `apply_*_plan` three-function interface (`reasons/api.py:4836`, `:4867` vs `:5079`, `:5107`, `:5139`).
- `user-can-change-kept-node-after-write` — The `keep` field is user-editable between write and parse, verified by `test_dedup_plan_user_can_edit_kept` (`tests/test_derive.py:882`).
