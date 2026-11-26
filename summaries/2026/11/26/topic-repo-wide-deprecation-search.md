---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search all file types, not just Markdown/Python, for `reasons_lib`, `compat shim`, `deprecat`, and `remove`.

# Searching All File Types for `reasons_lib`, `compat shim`, `deprecat`, and `remove`

## What the instruction asks for

Rather than limiting a repository search to the "obvious" source/doc extensions (`.py`, `.md`), the task is to grep *everything* — config files, generated manifests, tool/agent metadata, and version-control artifacts — for four related signals:

1. `reasons_lib` — the legacy/alback-compat package name.
2. `compat` (the "compat shim" concept) — backwards-compatibility re-exports.
3. `deprecat` — deliberate deprecation markers.
4. `remove` — removal/retraction operations in the API.

This matters because the backcompat story spans **packaging**, **generated metadata**, **agent tooling**, and **code** — not just `*.py` / `*.md`.

## What each pattern turned up

### `reasons_lib` — 17 matches, spread across 4 file types
- **`.md` docs**: `CHANGELOG.md:136` (mentions the `reasons_lib.api` layer), `CLAUDE.md:24` and `CLAUDE.md:35` ("`reasons_lib/` — backwards-compat shim — re-exports from `reasons/`").
- **`.toml` packaging**: `pyproject.toml:31` — `packages = ["reasons", "reasons_lib"]`. So the shim directory is actually **shipped as an importable package**, not just documentation of an alias.
- **`.json` generated manifest**: `.forge/proposed-entries.json:2-14` — 12 entries for `entries/2026/04/23/reasons_lib-*.md`, i.e., the shim is being indexed/documented by a downstream tooling process.
- **No `.py` hits** for `reasons_lib` itself — the actual shim modules live at `reasons_lib/__init__.py`, `reasons_lib/api.py`, etc., but their *contents* refer to `reasons`, not `reasons_lib` (they document what to use instead).

### `compat` (the "compat shim" signal) — 49 matches, mostly noise + concentrated shim
- **The shim itself**: every `reasons_lib/*.py` file starts with a docstring `"Backwards-compatibility shim — use ``from reasons.X`` instead."` — e.g., `reasons_lib/__init__.py:1`, `reasons_lib/api.py:1`, `reasons_lib/ask.py:1`, `reasons_lib/build_wiki.py:1`, `reasons_lib/check_stale.py:1`, `reasons_lib/cli.py:1`, `reasons_lib/cluster.py:1`, `reasons_lib/compact.py:1`, `reasons_lib/contradictions.py:1`, `reasons_lib/derive.py:1`, `reasons_lib/export_card.py:1`, `reasons_lib/export_markdown.py:1`, `reasons_lib/hf.py:1`, `reasons_lib/import_agent.py:1`, `reasons_lib/import_beliefs.py:1`. This is the "compat shim" phrase you were looking for — it lives in **docstrings**, not in docs.
- **A different meaning entirely**: the pattern `compat` also matches `beliefs.md-compatible` and `HuggingFace-compatible` — e.g., `README.md:184`, `reasons/export_markdown.py:1,8`, `reasons/export_card.py:1`, `reasons/api.py:2546`, `reasons/cli.py:3498`, `.claude/skills/reasons/SKILL.md:154`. These are **output-format compatibility**, not import-path compatibility. Important not to conflate them.
- **Deprecation-adjacent aliases**: `reasons/cli.py:3853` (`# research (backward-compatible alias for repair)`) and `reasons/ask.py:177` (`Backward-compat wrapper`).

### `deprecat` — only 3 matches, **all in code, none in docs**
- `reasons/api.py:3919` — `'outdated', 'deprecated', 'fragile', ...` (a heuristic keyword list for staleness/quality detection, *not* an API deprecation).
- `reasons/cli.py:3064` — `help="(deprecated alias for --domain-id)"`.
- `reasons/cli.py:3854` — `p = sub.add_parser("research", help="Alias for 'repair' (deprecated)")`.

This is the most important observation for a "search all file types" review: **there are two real deprecations in the CLI (`--domain-id` alias and `research` command), and neither is mentioned in `CHANGELOG.md` or `README.md`.**
The rename itself is documented in `CHANGELOG.md` / `CLAUDE.md` ("Previously called rms — renamed…"), but not as a *deprecation*, and the CLI aliases have no doc-level deprecation notice.

### `remove` — 130 matches, ~95% noise, but the signal is real
- **Version-control noise** (should generally be excluded from "search all files" in normal use):
  - `.git/hooks/pre-rebase.sample:49`, `.git/hooks/prepare-commit-msg.sample:12`, `.git/hooks/sendemail-validate.sample:75`
  - `.git/packed-refs:98-99` — branches `remove-justification`, `remove-update-command`
  - `.git/FETCH_HEAD:97-98` — same branches from `github.com:benthomasson/ftl-reasons`
  These reveal *feature branches* named after removals — useful history, but not "code" removal.
- **Real removal semantics in the API**:
  - `reasons/api.py:962` — `remove_tags(node_id, tags, ...)`; docstring/return shape `{"node_id": str, "tags": list[str], "removed": list[str]}`; raises `NotImplementedError` for PostgreSQL at line 966.
  - `reasons/api.py:972-984` — implementation of the removal loop (`removed.append(tag)`).
  - `reasons/api.py:1129` — `remove_justification(...)`; dispatches to Postgres at line 1135, otherwise calls `net.remove_justification(node_id, index)` at line 1138.
  - `reasons/cli.py:93-104` — `cmd_remove_justification(args)` prints `Removed:`/`Was:` lines sourced from the dict returned by `api.remove_justification`.
  - `reasons/cli.py:599-604` — `api.remove_tags` wiring + `Removed:` output.
  - `reasons/cli.py:940-941` — `result['beliefs_removed']` summary of a bulk retraction.
- **Same-word but different concept**: `reasons/api.py:2027` ("retracts removed ones" during import-merge), `reasons/api.py:519` (restoration from outlist when a blocker is removed), `reasons/contradictions.py:23,26` ("Incompatible properties" — `compat` inside a longer word, a false positive for the `compat` pattern too).

## Why "all file types" mattered here

| Non-`.py`/`.md` hit | What it proved |
|---|---|
| `pyproject.toml:31` | The shim is a **shipped package**, not just a doc note. Removing it would break `pip install ftl-reasons` for `import reasons_lib` users. |
| `.forge/proposed-entries.json:2-14` | Downstream tooling already has 12 wiki/docs entries keyed on `reasons_lib-*` — a rename would need a migration in that manifest. |
| `.claude/skills/reasons/SKILL.md:109,154` | An agent skill file references both the SL-justification removal caveat and the `beliefs.md-compatible` export. It's a consumer of the API surface, so it lives in scope. |
| `.git/packed-refs`, `.git/FETCH_HEAD` | Two long-lived branches (`remove-justification`, `remove-update-command`) indicate the `remove_justification` API has been actively worked on and that a `update` command was removed at some point. |

## What the observations don't cover

- The full `compat` result list (only ~30 of 49 cited) — remaining `reasons_lib/*.py` docstrings (e.g., `network.py`, `storage.py`, `mcp_server.py`, `merkle.py`) are almost certainly present but truncated.
- The full `remove` result set (only ~30 of 130 shown) — remaining hits likely include more `reasons/api.py` retraction paths and `tests/`.
- Whether `CHANGELOG.md` has an entry for the `research`→`repair` and `--domain-id` deprecations (nothing in the excerpt suggests it).
- Whether `.gitignore`-style exclusions (e.g., a `.rgignore` or search tool config) are what produced the `.git/` hits — implying the search tool *does not* exclude VCS dirs.

## Topics to Explore

- [file] `reasons_lib/` — The full set of shim modules; confirm every one re-exports `from reasons.<name>` and none contain logic.
- [function] `reasons/api.py:remove_justification` — The canonical removal/retraction entry point; compare SQLite path (`net.remove_justification`) with the Postgres dispatch at line 1135.
- [file] `CHANGELOG.md` — Reads the rename/backcompat history; check whether the `research` and `--domain-id` deprecations ever landed in an entry.
- [general] `deprecation-coverage-gap` — Reconcile the 3 `deprecat` code hits against doc mentions; likely needs a changelog/README addition.
- [general] `search-scope-hygiene` — Decide whether `.git/`, `.forge/`, and `.claude/` belong in default searches or should be ignored to reduce `remove` noise from 130→~35.

## Beliefs

- `reasons-lib-is-a-packaging-shim` — `reasons_lib` is declared in `pyproject.toml` (`packages = ["reasons", "reasons_lib"]`) and each module is a one-line re-export shim, so it must be shipped to preserve `import reasons_lib` compatibility.
- `compat-shim-is-documented-in-docstrings-not-docs` — The phrase "backwards-compatibility shim" appears only in the `reasons_lib/*.py` module docstrings, not in `README.md` or `CHANGELOG.md`.
- `compat-matches-conflate-two-concepts` — The `compat` pattern matches both import-path backward compatibility and output-format compatibility (`beliefs.md-compatible`, `HuggingFace-compatible`), which are unrelated concerns.
- `cli-deprecations-are-undocumented` — The only real `deprecat` markers (`reasons/cli.py:3064` for `--domain-id`, `reasons/cli.py:3854` for `research`) have no corresponding `deprecat` mention in `.md` files.
- `reasons-lib-has-downstream-forge-entries` — `.forge/proposed-entries.json` contains 12 entries keyed on `entries/2026/04/23/reasons_lib-*.md`, so any rename or removal of `reasons_lib` requires updating that manifest.
- `remove-pattern-is-dominated-by-git-noise` — Of 130 `remove` matches, the majority are `.git/hooks/*.sample`, `.git/packed-refs`, and `.git/FETCH_HEAD`; the semantic hits cluster in `reasons/api.py` (`remove_tags` at 962, `remove_justification` at 1129) and `reasons/cli.py` (`cmd_remove_justification` at 93).
