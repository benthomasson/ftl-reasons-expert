---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Look for roadmap or issue references that mention deprecating or removing the compatibility package.

## What the observations show

The compatibility package in this project is `reasons_lib`. `CLAUDE.md:35` explicitly describes it as a “backwards-compat shim — re-exports from `reasons/`”, and `CLAUDE.md:24` says `reasons` is “also aliased as `reasons_lib` for backwards compat”. `pyproject.toml:31` still ships it:

```toml
packages = ["reasons", "reasons_lib"]
```

So the package is currently present and packaged, not removed.

The requested search was for roadmap or issue references that mention deprecating or removing that compatibility package. Based on the supplied observations:

- The Markdown search for `roadmap|issue|TODO|FIXME` returned `total_count: 0`.
- The Markdown search for `deprecat` returned `total_count: 0`.
- The only Markdown mentions of `reasons_lib` are historical/descriptive:
  - `CHANGELOG.md:136` — mentions `reasons_lib.api` as the Python API layer.
  - `CLAUDE.md:24` — mentions `reasons_lib` as a backwards-compat alias.
  - `CLAUDE.md:35` — describes `reasons_lib/` as a backwards-compat shim.
- None of those mentions say the package is deprecated, scheduled for removal, or tracked by a roadmap/issue.
- The Python `deprecat` matches are unrelated:
  - `reasons/api.py:3919` — a word list containing `'deprecated'` alongside `'outdated'`, `'fragile'`, etc.
  - `reasons/cli.py:3064` — `--domain-id` deprecated alias.
  - `reasons/cli.py:3854` — `research` alias for `repair` marked deprecated.
- The Markdown `compatib` matches are also unrelated: they refer to `beliefs.md`-compatible export, not package compatibility:
  - `.claude/skills/reasons/SKILL.md:154`
  - `CLAUDE.md:73`
  - `README.md:184`

**Conclusion:** The provided observations contain **no roadmap or issue references that mention deprecating or removing the compatibility package**. The evidence instead shows `reasons_lib` is documented and packaged as an active backwards-compat shim. To prove absence repo-wide, you would need broader searches outside the supplied Markdown/Python sweeps, such as a full-repo grep for `reasons_lib`, “compatibility package”, “compat shim”, “deprecat”, “remove”, plus a GitHub issue/PR search.

## Topics to Explore

- [file] `CLAUDE.md` — Documents `reasons_lib` as a backwards-compat shim and alias for `reasons`.
- [file] `pyproject.toml` — Confirms `reasons_lib` is still included in the setuptools package list.
- [general] `repo-wide-deprecation-search` — Search all file types, not just Markdown/Python, for `reasons_lib`, `compat shim`, `deprecat`, and `remove`.
- [general] `github-issue-search` — Search the issue/PR tracker for references to `reasons_lib` or compatibility-package removal.
- [file] `CHANGELOG.md` — Check later changelog entries for any future removal/deprecation note about `reasons_lib.api`.

## Beliefs

- `reasons-lib-is-backwards-compat-shim` — `CLAUDE.md:35` describes `reasons_lib/` as a backwards-compat shim that re-exports from `reasons/`.
- `reasons-lib-is-included-in-package-list` — `pyproject.toml:31` includes both `"reasons"` and `"reasons_lib"` in `[tool.setuptools]` `packages`.
- `no-markdown-deprecation-matches` — The Markdown search for `deprecat` returned zero matches.
- `no-literal-markdown-roadmap-issue-markers` — The Markdown search for `roadmap|issue|TODO|FIXME` returned zero matches.
- `python-deprecation-matches-do-not-mention-reasons-lib` — The three `deprecat` Python matches at `reasons/api.py:3919` and `reasons/cli.py:3064,3854` do not reference `reasons_lib`.
