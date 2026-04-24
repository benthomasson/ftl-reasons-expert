# CLAUDE.md

## What This Is

A code-expert knowledge base for [ftl-reasons](https://github.com/benthomasson/ftl-reasons). Contains exploration entries, beliefs, code reviews, and SDLC logs produced by analyzing the ftl-reasons TMS implementation.

## Key Files

- `beliefs.md` / `network.json` -- portable belief exports (do not edit directly; use `reasons` CLI)
- `reasons.db` -- SQLite belief store (gitignored; regenerate with `reasons import-json network.json`)
- `entries/` -- exploration entries written by code-expert
- `reviews/` -- multi-model code review reports
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

## Conventions

- Belief IDs are kebab-case (`add-nogood-always-records`)
- Belief types: OBSERVATION (from code), DERIVED (from other beliefs), GATE (blocked by defects)
- Entries live under `entries/YYYY/MM/DD/`
- Reviews live under `reviews/<repo-pr-number>/`
- SDLC artifacts are archived in `logs/`

## Target Codebase

The target is [benthomasson/ftl-reasons](https://github.com/benthomasson/ftl-reasons) at `/Users/ben/git/ftl-reasons`. It implements Doyle's 1979 Truth Maintenance System: nodes with SL justifications, outlist-based non-monotonic reasoning, retraction cascades, nogood detection, and multi-agent federation.
