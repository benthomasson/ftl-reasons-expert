# ftl-reasons-expert

A code-expert knowledge base for [ftl-reasons](https://github.com/benthomasson/ftl-reasons), a Python implementation of Doyle's 1979 Truth Maintenance System (TMS).

This repo contains structured exploration entries, extracted beliefs, multi-model code reviews, and SDLC pipeline logs produced by analyzing the ftl-reasons codebase with [code-expert](https://github.com/benthomasson/ftl-code-expert).

## Belief State

- **122 IN** beliefs (held as true)
- **55 OUT** beliefs (retracted or defeated)
- Beliefs are tracked in `reasons.db` (gitignored) and exported to `beliefs.md` and `network.json`

Beliefs range from low-level observations ("add_nogood always records") to high-level derived conclusions ("the TMS architecture exhibits structural duality between local and federated reasoning"). OUT beliefs include defects that were fixed via PRs and merged back into ftl-reasons.

## Contents

```
beliefs.md              # Portable belief registry (markdown)
network.json            # Lossless belief network export (JSON)
proposed-beliefs.md     # Beliefs proposed but not yet accepted
.code-expert/           # Scanner config, topic queue, proposed entries
entries/2026/04/23/     # 17 exploration entries covering:
                        #   - Core modules (network, storage, api, cli)
                        #   - Key algorithms (propagation, justification, nogood resolution)
                        #   - Design patterns (outlist semantics, multi-agent federation)
                        #   - Doyle 1979 TMS theory
reviews/                # 18 multi-model code review reports (Claude + Gemini)
logs/                   # 8 SDLC pipeline artifact archives
```

## Workflow

This knowledge base was built using the following pipeline:

1. **Scan** -- `code-expert scan` identified key files and populated the topic queue
2. **Explore** -- `code-expert explore --loop` generated detailed entries for each topic
3. **Extract beliefs** -- `code-expert propose-beliefs` then `code-expert accept-beliefs`
4. **Derive** -- `reasons derive --exhaust` built multi-round reasoning chains via LLM
5. **Identify defects** -- OUT beliefs with active blockers were identified as potential bugs
6. **File issues** -- `code-expert file-issues` created GitHub issues on ftl-reasons
7. **Fix** -- `ftl-sdlc-loop` generated PRs, `code-review` validated them
8. **Merge and retract** -- `ftl-merge --auto-retract` merged PRs and retracted fixed-defect beliefs

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
