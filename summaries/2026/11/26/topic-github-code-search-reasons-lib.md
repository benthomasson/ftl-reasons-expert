---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search downstream repos and issue bodies for real `import reasons_lib` / `from reasons_lib` statements.

# Search Downstream Repos and Issue Bodies for Real `import reasons_lib` / `from reasons_lib` Statements

## What the task asks for

The objective is an **external evidence sweep**: find *other* repositories (downstream consumers) and *issue/PR bodies* that contain actual Python import statements referencing the legacy `reasons_lib` alias — i.e. literal `import reasons_lib`, `from reasons_lib import ...`, `from reasons_lib.api import ...`, etc. This matters because `reasons_lib` is documented as a backwards-compatibility shim (CLAUDE.md line 35: `reasons_lib/  # backwards-compat shim — re-exports from reasons/`) and is still shipped (pyproject.toml line 31: `packages = ["reasons", "reasons_lib"]`). Knowing whether anyone actually imports it decides whether the shim can be deleted.

## What the observations actually contain

**No downstream search was performed.** Every artifact in the observation set is *internal to this repo*:

| Observation | What it shows | Relevance to the task |
|---|---|---|
| `grep_downstream_mentions` | 1 hit, `reasons/report_gated.py:24`, text `"...and its downstream impact (number of \"` | Unrelated prose in a report generator — "downstream impact", not downstream repos |
| `grep_issue_body_mentions` | **0 matches** for `issue bod(y\|ies)` | The repo doesn't even discuss issue bodies; no issue bodies were fetched |
| `grep_reasons_lib_mentions` | 17 hits, all in `CHANGELOG.md`, `CLAUDE.md`, `pyproject.toml`, `.forge/proposed-entries.json` | All are docs/config/diary metadata, **zero are import statements** |
| `list_claude_skills` | `.claude/skills/reasons/SKILL.md` | Local skill definition only |
| `read_forge_report` | `.forge/report.json` pipeline `diary-update` with steps `propose-beliefs` / `accept-beliefs` | The forge pipeline runs belief proposal, **not** any repo/issue search step |

### Why the 17 `reasons_lib` hits are not evidence of usage

- **`CLAUDE.md:24`** — `reasons/  # importable as \`import reasons\` (also aliased as reasons_lib for backwards compat)`. This is a *narrative claim about intent*, not a call site.
- **`CLAUDE.md:35`** — declares `reasons_lib/` a shim. Still self-description.
- **`pyproject.toml:31`** — packaging metadata: the shim is *exported*, which says nothing about whether it is *imported* elsewhere.
- **`CHANGELOG.md:136`** — `- Python API layer (\`reasons_lib.api\`) returning dicts for CLI and tool-call consumers`. This is the strongest hint of real consumers, but it names an internal module path, not an external importer.
- **`.forge/proposed-entries.json` (lines 2–14)** — 13 entries keyed by `reasons_lib` *module paths* (e.g. `entries/2026/04/23/reasons_lib-network-challenge.md`). Note these are **derived from the local `reasons_lib` package layout** (a diary generator enumerating modules), not harvested from any GitHub code search. The remaining 4 entries in the same file are unrelated (`scan-ftl-reasons`, `topic-doyle-1979-tms`, `topic-multi-agent-federation`, `topic-outlist-semantics`).

### Verdict

The observations are **insufficient** to answer the question and appear to record an attempted grep of the local checkout rather than a search of downstream repos. What is missing:

1. **A list of downstream repos** (org-scoped or via `gh repo list`, or a dependency-graph / reverse-dependency query).
2. **GitHub code-search results** for `"import reasons_lib"` and `"from reasons_lib"` (code search treats these as phrase queries; `from reasons_lib` will not match `import reasons_lib`, so both patterns are needed, plus `reasons_lib.` for attribute-style usage).
3. **Issue/PR body fetch output** — `grep_issue_body_mentions` returning 0 only proves the *local repo text* never says "issue body"; it is not a proxy for `gh search issues "reasons_lib"`.
4. **The contents of `reasons_lib/__init__.py`** — never read here, so we cannot confirm it is a pure re-export shim versus containing its own logic (relevant if the alias is load-bearing).
5. **Minor data quality flag:** `read_claude_md` reports `total_lines: 99` but `lines_returned: 98` with content starting on a blank line, so offsets in the returned blob are shifted by roughly one relative to the grep-reported line numbers (24, 35). Trust the grep line numbers.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Confirm the shim is a pure re-export (`from reasons import *` style) and has no independent logic, which determines whether removal is safe
- [repo] `ftl-reasons` — The distributed package name from `pyproject.toml` line 5; use it to query reverse-dependencies and downstream importers on PyPI/GitHub
- [general] `github-code-search` — Run `gh search code "from reasons_lib" --filename=*.py` across the org; code search does not match `import reasons_lib` with the same query, so run both patterns
- [general] `issue-body-search` — Run `gh search issues "reasons_lib"` to actually cover the issue/PR-body half of the objective, which no current observation touches
- [file] `reasons/report_gated.py` — Inspect line 24 to confirm the sole "downstream" hit is unrelated report prose and not, say, a repo-scanning helper

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — CLAUDE.md line 35 documents `reasons_lib/` as a shim that re-exports from `reasons/`, not an independent implementation.
- `pyproject-packages-contains-reasons-lib` — `pyproject.toml` line 31 lists `packages = ["reasons", "reasons_lib"]`, so the shim is still distributed in version 0.58.0.
- `changelog-maps-reasons-lib-api-to-dict-returning-layer` — `CHANGELOG.md` line 136 attributes `reasons_lib.api` to the dict-returning Python API layer for CLI and tool-call consumers.
- `reasons-lib-mentions-are-all-in-repo-metadata` — All 17 `reasons_lib` hits are in `CHANGELOG.md`, `CLAUDE.md`, `pyproject.toml`, or `.forge/proposed-entries.json`; none is a Python import statement.
- `forge-entries-map-to-reasons-lib-module-paths` — 13 of the 17 keys in `.forge/proposed-entries.json` are derived from local `reasons_lib` module paths, evidencing local enumeration rather than downstream discovery.
- `downstream-grep-hit-is-unrelated-to-repos` — The single `downstream` match is `reasons/report_gated.py:24` describing report impact, not repository search.
