# ftl-reasons-expert

A code-expert knowledge base for [ftl-reasons](https://github.com/benthomasson/ftl-reasons), a Python implementation of Doyle's 1979 Truth Maintenance System (TMS).

This repo contains structured exploration entries, extracted beliefs, multi-model code reviews, and SDLC pipeline logs produced by analyzing the ftl-reasons codebase with [code-expert](https://github.com/benthomasson/ftl-code-expert).

## Belief State

- **596 IN** beliefs (held as true) / **65 OUT** (retracted or defeated) — 661 total
- **313 premises** (direct observations from code) / **283 derived** (reasoned from other beliefs)
- **10 retracted premises** — defects fixed via PRs merged back into ftl-reasons
- **0 active blockers** — all GATE beliefs resolved
- **Max derivation depth: 15** — 16 beliefs at depth 11+
- Beliefs are tracked in `reasons.db` (gitignored) and exported to `beliefs.md` and `network.json`

### Derivation Depth Distribution

| Depth | Count | Examples |
|-------|-------|---------|
| 0 | 313 | Premises: `sl-justification-semantics`, `propagation-is-bfs`, `access-control-enforced-at-read-not-write` |
| 1-4 | 177 | `propagation-is-safe-and-terminating`, `tms-core-is-crash-safe`, `ask-is-fault-tolerant-and-bounded` |
| 5-10 | 90 | `invariant-preservation-is-comprehensive`, `external-beliefs-achieve-integration-parity` |
| 11-15 | 16 | `system-guarantees-are-universal-permanent-and-verifiable` (apex, depth 15) |

### Key Derived Beliefs

Selected beliefs illustrating the derivation chain from system-level conclusions down to code observations:

**Apex** (depth 15):
- `system-guarantees-are-universal-permanent-and-verifiable` — The system's guarantees are simultaneously universal (all belief types), permanent (no temporal degradation), and verifiable (independently auditable through the maintenance loop)

**Self-maintenance** (depth 8-12):
- `system-is-fully-characterized-self-maintaining-loop` — Closed loop characterized along origin-agnosticism, observability, and self-sustainability
- `self-correction-has-complete-traceable-history` — Temporally complete and historically traceable across creation-time and maintenance-time
- `invariant-preservation-is-comprehensive` — Invariants preserved through closed revision/lifecycle loop and structural/dynamic enforcement

**Architecture and integration** (depth 5-8):
- `all-mutations-preserve-integrity-under-adverse-conditions` — Every structural modification preserves integrity even under adverse graph conditions
- `external-beliefs-achieve-integration-parity` — External beliefs achieve full parity with internal beliefs across lifecycle management and deterministic revision
- `architecture-is-self-contained-and-safely-layered` — Zero runtime dependencies externally, clean three-layer boundaries internally

**Revision and contradiction** (depth 3-6):
- `contradiction-resolution-is-traceable-and-recoverable` — Minimizes disruption with guided recovery and maintains consistent artifact identification
- `belief-revision-is-comprehensive-and-minimal` — All revision handled through two minimal mechanisms: the outlist primitive and least-entrenched culprit selection
- `revision-is-universally-safe` — Every belief, including all semantic edge cases, can be revised through either reactive or proactive paths

**Core safety** (depth 1-3):
- `retraction-cascade-is-transitive-and-terminating` — Cascades propagate OUT to all transitively dependent nodes and are guaranteed to terminate safely
- `propagation-is-safe-and-terminating` — BFS prevents stack overflow, stop-on-unchanged prevents oscillation, retracted nodes are skipped
- `tms-core-is-crash-safe` — Deterministic termination, pure evaluation, and conservative failure semantics ensure correct results across all reachable nodes

## Contents

```
beliefs.md              # Portable belief registry (markdown)
network.json            # Lossless belief network export (JSON)
reasons.db              # SQLite belief store (gitignored)
proposed-beliefs.md     # Beliefs proposed but not yet accepted
.code-expert/           # Scanner config, topic queue, proposed entries
entries/                # 75 exploration entries covering:
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
