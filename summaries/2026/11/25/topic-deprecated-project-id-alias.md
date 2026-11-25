---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Whether and where `--project-id` / `REASONS_PROJECT_ID` are meant to be withdrawn; a search for `project_id` in `reasons/` reveals every use site.

## Conclusion

Based on the supplied observations, there is **no evidence that `--project-id` or `REASONS_PROJECT_ID` is being withdrawn, deprecated, or migrated**. More strongly: the supplied searches found **zero occurrences** of either identifier in `reasons/`, `reasons_lib/`, tests, or docs/changelog.

In practical terms:

- `--project-id` is not present in the first 500 lines of `reasons/cli.py`, and the directory-wide search over `reasons/` returned `total_count: 0`.
- `REASONS_PROJECT_ID` is likewise absent from the searched code.
- No withdrawal/deprecation terms such as `withdraw`, `deprecat`, `legacy`, or `obsolete` appear in `reasons/`.
- `CHANGELOG.md` lines 1–139 contain no mention of `project-id`, `REASONS_PROJECT_ID`, `withdraw`, or `deprecat`.

So the answer is: **there is no current in-code withdrawal site, because there is no current in-code use site.** If this option existed historically, its removal is not documented in the observed changelog. The only active environment-variable pattern visible in the CLI excerpt is `REASONS_PG_CONNINFO`, used by several PostgreSQL guard paths in `reasons/cli.py`.

### Evidence from observations

| Observation | Result |
|---|---|
| `project_id_use_sites_in_reasons` | `matches: []`, `total_count: 0` |
| `withdrawal_or_deprecation_terms_in_reasons` | `matches: []`, `total_count: 0` |
| `project_id_use_sites_in_reasons_lib` | `matches: []`, `total_count: 0` |
| `project_id_in_tests` | `matches: []`, `total_count: 0` |
| `project_id_docs_changelog` | `matches: []`, `total_count: 0` |
| `reasons/cli.py` lines 1–500 | No `project-id` / `REASONS_PROJECT_ID` in the returned excerpt |
| `CHANGELOG.md` lines 1–139 | No withdrawal/deprecation/project-id references |

### What is not fully covered

The first 500 of 4015 lines of `reasons/cli.py` were returned, so that excerpt alone does not prove the entire CLI has no dynamic or later-defined project-id flag. However, the explicit `reasons/` directory search returned zero matches for `project[-_]id|REASONS_PROJECT_ID`, which is stronger evidence for that directory.

The observations do not include searches of files outside `reasons/`, `reasons_lib/`, tests, and docs/changelog, such as `pyproject.toml`, README files, packaging metadata, or external scripts. Those would need separate inspection if you need repository-wide certainty.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect lines 501–4015 to confirm no later argparse definition dynamically adds `--project-id`.
- [function] `reasons/cli.py:_backend_kwargs` — Verify how CLI arguments are translated into backend kwargs and whether any legacy project-scoping config passes through it.
- [general] `reasons-lib-api-surface` — Check whether `reasons_lib.api` exposes project-scoped parameters independently of the CLI.
- [general] `environment-variable-usage` — Audit all `os.environ` reads, especially `REASONS_PG_CONNINFO`, to separate active configuration from removed legacy variables.
- [file] `CHANGELOG.md` — Confirm release history and whether a historical project-id removal is undocumented.

## Beliefs

- `reasons-has-no-project-id-use-sites` — The search for `project[-_]id|REASONS_PROJECT_ID` in `reasons/` returned zero matches.
- `reasons-has-no-withdrawal-markers` — The search for withdrawal/deprecation terms in `reasons/` returned zero matches.
- `reasons-lib-has-no-project-id-use-sites` — The same project-id pattern returned zero matches in `reasons_lib/`.
- `tests-have-no-project-id-use-sites` — The same project-id pattern returned zero matches in tests.
- `changelog-has-no-project-id-deprecation-entry` — `CHANGELOG.md` lines 1–139 contain no `project-id`, `REASONS_PROJECT_ID`, `withdraw`, or `deprecat` match.
