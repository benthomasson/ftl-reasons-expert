# CLAUDE.md

## What This Is

A code-expert knowledge base for [ftl-reasons](https://github.com/benthomasson/ftl-reasons) — a Python TMS (Truth Maintenance System) implementation. Contains 17 exploration entries, 189 beliefs (130 IN / 59 OUT), 18 code reviews, and 8 SDLC pipeline logs.

All 10 defect premises have been retracted and their GATE beliefs resolved. Zero active blockers remain.

## Key Files

- `beliefs.md` / `network.json` -- portable belief exports (do not edit directly; use `reasons` CLI)
- `reasons.db` -- SQLite belief store (gitignored; regenerate with `reasons import-json network.json`)
- `entries/` -- exploration entries written by code-expert
- `reviews/` -- multi-model code review reports (Claude + Gemini)
- `proposed-beliefs.md` -- beliefs awaiting review

## Working with Beliefs

```bash
reasons status                    # show all beliefs
reasons explain <belief-id>       # trace why IN or OUT
reasons search "query"            # full-text search
reasons export-markdown           # regenerate beliefs.md
reasons export                    # regenerate network.json
```

## Working with Code Expert

```bash
code-expert status                # dashboard
code-expert topics                # see exploration queue
code-expert explore               # explore next topic
code-expert propose-beliefs       # extract beliefs from entries
code-expert accept-beliefs        # import accepted beliefs into reasons.db
code-expert derive --auto         # LLM-driven derivation chains
code-expert file-issues --dry-run # preview issues for OUT beliefs with blockers
```

## Updating Beliefs After Merges

```bash
# After merging PRs that fix defects:
ftl-merge <PR-numbers> --auto-retract -e ~/git/ftl-reasons-expert -c ~/git/ftl-reasons
# Or manually:
reasons retract <belief-id> --reason "Fixed in PR #N"
reasons export-markdown -o beliefs.md
reasons export > network.json
```

## Conventions

- Belief IDs are kebab-case (`add-nogood-always-records`)
- Premises: direct observations from code (no justifications)
- Derived: reasoned from other beliefs via SL/CP justifications
- GATE beliefs: derived beliefs with outlist conditions (flip IN when blockers go OUT)
- Entries live under `entries/YYYY/MM/DD/`
- Reviews live under `reviews/<repo-pr-number>/`
- SDLC artifacts are archived in `logs/`

## Target Codebase

The target is [benthomasson/ftl-reasons](https://github.com/benthomasson/ftl-reasons) at `/Users/ben/git/ftl-reasons`. It implements Doyle's 1979 Truth Maintenance System: nodes with SL justifications, outlist-based non-monotonic reasoning, retraction cascades, nogood detection, and multi-agent federation.

## Known Issue

Outlist nodes are not tracked in the dependents index, so GATE beliefs are not automatically re-evaluated when outlist nodes change truth value. After retracting defect premises, GATE beliefs may need manual `reasons assert` to flip IN.
