# ftl-reasons-expert

A code-expert knowledge base for [ftl-reasons](https://github.com/benthomasson/ftl-reasons), a Python implementation of Doyle's 1979 Truth Maintenance System (TMS).

This repo contains structured exploration entries, extracted beliefs, multi-model code reviews, and SDLC pipeline logs produced by analyzing the ftl-reasons codebase with [code-expert](https://github.com/benthomasson/ftl-code-expert).

## Belief State

- **241 IN** beliefs (held as true) / **59 OUT** (retracted or defeated) — 300 total
- **24 premises** (direct observations from code) / **217 derived** (reasoned from other beliefs)
- **10 retracted premises** — defects fixed via PRs merged back into ftl-reasons
- **0 active blockers** — all GATE beliefs resolved
- **Max derivation depth: 14** — 16 beliefs at depth 11+, 2 terminal beliefs at depth 14
- Beliefs are tracked in `reasons.db` (gitignored) and exported to `beliefs.md` and `network.json`

Beliefs range from low-level code observations ("add-nogood-always-records") through mid-level architectural properties ("revision-is-universally-safe", "external-beliefs-achieve-integration-parity") to the depth-14 apex belief `system-guarantees-are-universal-permanent-and-verifiable` — the conclusion that the system's guarantees extend to all belief types, hold indefinitely, and are independently auditable.

### Derivation Depth Distribution

| Depth | Count | Examples |
|-------|-------|---------|
| 0 | 24 | Premises: `sl-justification-semantics`, `propagation-is-bfs` |
| 1-4 | 120 | `revision-is-universally-safe`, `tms-handles-all-conditions-safely` |
| 5-10 | 95 | `self-correction-has-complete-traceable-history`, `invariant-preservation-is-comprehensive` |
| 11-14 | 16 | `system-guarantees-are-universal-permanent-and-verifiable` (apex) |

## Contents

```
beliefs.md              # Portable belief registry (markdown)
network.json            # Lossless belief network export (JSON)
reasons.db              # SQLite belief store (gitignored)
proposed-beliefs.md     # Beliefs proposed but not yet accepted
.code-expert/           # Scanner config, topic queue, proposed entries
entries/                # 18 exploration entries covering:
                        #   - Core modules (network, storage, api, cli)
                        #   - Key algorithms (propagation, justification, nogood resolution)
                        #   - Design patterns (outlist semantics, multi-agent federation)
                        #   - Doyle 1979 TMS theory
                        #   - Defect resolution writeup
reviews/                # 18 multi-model code review reports (Claude + Gemini)
                        #   PRs #5-#7, #12-#15, #18, #27-#35, #42 on ftl-reasons
logs/                   # 8 SDLC pipeline artifact archives
```

## Workflow

This knowledge base was built using the following pipeline:

1. **Scan** -- `code-expert scan` identified key files and populated the topic queue
2. **Explore** -- `code-expert explore --loop` generated detailed entries for each topic
3. **Extract beliefs** -- `code-expert propose-beliefs` then `code-expert accept-beliefs`
4. **Derive** -- `reasons derive --exhaust` built multi-round reasoning chains via LLM
5. **Identify defects** -- GATE beliefs with active blockers were identified as potential bugs
6. **File issues** -- `code-expert file-issues` created GitHub issues on ftl-reasons
7. **Fix** -- `ftl-sdlc-loop` generated PRs, `code-review` validated them with multi-model consensus
8. **Merge and retract** -- `ftl-merge --auto-retract` merged PRs and retracted fixed-defect beliefs
9. **Verify** -- GATE beliefs flipped IN via TMS propagation, confirming fixes resolved the defects

## Issues Fixed

All 10 defect premises were retracted after their fixes merged:

| Issue | Defect Premise | PR |
|-------|---------------|-----|
| [#22](https://github.com/benthomasson/ftl-reasons/issues/22) | `propagate-assumes-dependents-exist` | [#27](https://github.com/benthomasson/ftl-reasons/pull/27) |
| [#23](https://github.com/benthomasson/ftl-reasons/issues/23) | `derive-agent-count-bug` | [#27](https://github.com/benthomasson/ftl-reasons/pull/27) |
| [#24](https://github.com/benthomasson/ftl-reasons/issues/24) | `dependents-bidirectional-index`, `dependents-index-derived-on-load`, `dependents-is-manual-reverse-index` | [#31](https://github.com/benthomasson/ftl-reasons/pull/31) |
| [#25](https://github.com/benthomasson/ftl-reasons/issues/25) | `missing-source-file-is-silent` | [#32](https://github.com/benthomasson/ftl-reasons/pull/32) |
| [#26](https://github.com/benthomasson/ftl-reasons/issues/26) | `nogood-ids-assume-append-only` | [#33](https://github.com/benthomasson/ftl-reasons/pull/33) |
| [#36](https://github.com/benthomasson/ftl-reasons/issues/36) | `compact-token-estimate-is-word-count`, `compact-budget-only-limits-in-nodes` | [#39](https://github.com/benthomasson/ftl-reasons/pull/39) |
| [#37](https://github.com/benthomasson/ftl-reasons/issues/37) | `hash-truncation-is-16-hex` | [#40](https://github.com/benthomasson/ftl-reasons/pull/40) |
| [#38](https://github.com/benthomasson/ftl-reasons/issues/38) | (feature) access_tags for data source provenance | [#42](https://github.com/benthomasson/ftl-reasons/pull/42) |

## Tools

- [code-expert](https://github.com/benthomasson/ftl-code-expert) -- builds expert knowledge bases from codebases
- [ftl-reasons](https://github.com/benthomasson/ftl-reasons) -- TMS-based belief tracking (the target codebase)
- [multi-model-code-review](https://github.com/benthomasson/multi-model-code-review) -- Claude + Gemini code review
- [ftl-sdlc-loop](https://github.com/benthomasson/ftl-sdlc-loop) -- multi-agent SDLC pipeline
- [ftl-merge](https://github.com/benthomasson/ftl-merge) -- PR merge with belief auto-retraction

## Using This Knowledge Base

To work with the beliefs interactively (requires [ftl-reasons](https://github.com/benthomasson/ftl-reasons)):

```bash
# Import the portable exports into a local reasons.db
reasons init
reasons import-json network.json

# Query beliefs
reasons status
reasons explain <belief-id>
reasons trace <belief-id>
reasons search "propagation"

# Continue exploration
code-expert explore
code-expert propose-beliefs
```

## License

MIT
