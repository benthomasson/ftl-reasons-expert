---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Search git history / issue tracker for #16, since the current grep of `reasons/` yields no in-code marker.

## What the observations actually show

The stated goal is to search **git history / issue tracker** for `#16`. But the gathered observations are *filesystem greps only* — there is no `git log --grep`, no `gh issue view`, and no PR diff in the data. So we can reconstruct the paper trail for issue #16 from checked-in files, but we cannot yet see the commit SHA, author, merge commit, or issue body.

### Where `#16` does appear

The repo-wide search for `#16\b` (`search_repo_for_hash16`) found exactly two hits:

| File | Line | Text |
|---|---|---|
| `CHANGELOG.md` | 33 | `- Import-agent: moved active premise from antecedent to outlist so per-belief retraction works independently (#16, PR #17)` |
| `tests/test_import_agent.py` | 358 | `This is the core bug from issue #16 — active premise as antecedent` |

Note the initial match text was truncated to `( #16, PR #1`) but the full `read_changelog` output confirms it is **PR #17**.

### Why the `reasons/` grep is empty

The other searches corroborate that issue references live *outside* the `reasons/` package:

- `search_repo_for_issue16_variants` (`issue|bug|ticket|gh 16`, `closes/fixes #16`) → **0 matches.**
- `search_for_todo_refs_to_16` (`TODO/FIXME/... 16`) → **0 matches.**
- `inspect_issue_number_test_convention` (`issue\d+|issue[_-]\d+|#\d+`) → **0 matches.** This is the strongest signal that this pattern scan was scoped to `reasons/` source — otherwise it would have hit `#16` inside `CHANGELOG.md` and `tests/test_import_agent.py`, which the `#16\b` search already found.
- `search_workflows_for_issue_refs` → **0 matches** in `.github/workflows`, so CI doesn't reference #16 either.

Inference: there is genuinely **no in-code marker in `reasons/`** for issue #16. The fix was tracked through the changelog and a regression test rather than a code comment or TODO.

### What the CHANGELOG tells us

`CHANGELOG.md` (140 lines in full) places the fix under:

```
## [0.16.0] - 2026-04-18
### Fixed
- Import-agent: moved active premise from antecedent to outlist so per-belief retraction works independently (#16, PR #17)
- `_retracted` flag now set even when node is already OUT
- Internal metadata keys (`_retracted`, etc.) filtered from JSON export
```

So the timeline pointer is: **issue #16 → PR #17 → release 0.16.0 (2026-04-18)**. The behavioral change is localized to the import-agent logic — the "active premise" was being attached to the antecedent when it should have gone to the outlist, so that retracting an imported belief doesn't invalidate the others.

### What's missing

To fully answer "search git history / issue tracker," the following are **not** in the observations and need to be gathered:

1. `git log --all --grep='#16'` (and `--grep='Fixes #16'`, `--grep='Closes #16'`) to get commit SHAs, author, and the merge commit for PR #17.
2. `gh issue view 16` / tracker export to confirm the issue body, labels, and linked PR.
3. The PR #17 diff itself, to verify the change is only in `reasons_lib/import_agent.py` and consider whether a code-level `#16` comment should be added.
4. Whether `.forge/proposed-entries.json` (17 entries, 260 proposals, 0 accepted per `.forge/report.json`) contains an unrelated diary entry mentioning #16 — it does not in the hashes shown, but the entries themselves weren't read.

The next concrete step is to run `git log --all --grep='#16'` and `gh pr view 17 --repo <owner>/ftl-reasons` to close the loop.

## Topics to Explore

- [file] `CHANGELOG.md` — Line 33 anchors issue #16 to PR #17 and release 0.16.0; the canonical timeline pointer.
- [file] `tests/test_import_agent.py` — Line 358 is the regression test for "active premise as antecedent"; inspect surrounding test to see exact assertion on the outlist.
- [general] `git-log-grep-16` — Run `git log --all --grep='#16' --grep='Fixes #16' --grep='Closes #16'` to recover commit SHA, author, and PR #17 merge commit.
- [general] `github-issue-16` — Fetch the issue body and labels via `gh issue view 16`; confirms the reported symptom beyond the test comment.
- [general] `pr-17-diff` — The actual patch for the import-agent active-premise fix; verify scope is confined to `reasons_lib/import_agent.py`.

## Beliefs

- `issue-16-maps-to-pr-17` — `CHANGELOG.md` line 33 pairs issue #16 with PR #17 under release 0.16.0 (2026-04-18).
- `reasons-package-contains-no-issue-16-marker` — Repo-wide `#16\b` matches occur only in `CHANGELOG.md` and `tests/test_import_agent.py`; `reasons/` contains zero.
- `issue-16-fix-is-import-agent-active-premise` — The fix moves the active premise from antecedent to outlist so per-belief retraction is independent.
- `test-import-agent-regression-covers-issue-16` — `tests/test_import_agent.py:358` documents issue #16 as the "core bug" and exercises the active-premise-as-antecedent path.
- `changelog-is-the-only-durable-issue-reference` — No TODO/FIXME/XXX/HACK, workflow, or commit-message marker for #16 exists in tracked files.
