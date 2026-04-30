# Session Summary — 2026-04-29 (evening)

## Work completed

### PR #54: `list-negative` command (merged)
Added LLM-classified negative belief detection. Uses keyword pre-filter (~50 terms) then `claude -p` classification to find IN beliefs that describe problems, defects, or risks. Robust JSON parsing via `re.finditer` handles prose preamble and multi-line responses. 10 tests covering edge cases including prompt injection resistance.

### PR #56: `what_if_retract` and `what_if_assert` for PgApi (merged)
Transaction + ROLLBACK simulation pattern with `try/finally` for guaranteed rollback. BFS cascade depth tracking follows both antecedents and outlist via `_find_dependents`. 10 tests including outlist restoration verification.

### PR #62: Route `cmd_propagate` through API layer (merged, fixes #59)
`cmd_propagate` was the only CLI handler bypassing the API. Added `api.propagate()` using the standard `_with_network` pattern. CLI now delegates cleanly. 2 new API tests + 2 existing CLI tests pass.

### PR #71: Referential integrity for PgApi JSONB references (merged, fixes #61)
Added `_validate_refs` that checks all antecedent/outlist node IDs exist in `rms_nodes` before inserting justifications. Previously phantom references silently defaulted to OUT. 7 tests covering phantom antecedents, phantom outlist, multiple missing refs, valid refs, and premises.

### Issue #58: `outlist-nodes-not-in-dependents-index` retracted as stale
Discovered the blocker was already fixed in PR #31. Retracted the belief in the expert repo, ungating 8 derived beliefs.

### Issue #60: PgApi coverage gaps broken into sub-issues
23 missing PgApi functions catalogued and filed as 8 focused sub-issues (#63-#70) by category: simple methods, namespace, export, import, maintenance, derive, deduplication, repo/list_negative. Parent issue closed.

## Code expert pipeline
Ran `code-expert update` with `--sample` and `--timeout 600`. Produced 242 new beliefs (total 542), 217 derived. Analyzed gated and negative beliefs, filed issues #58-#61 for all 4 blockers.

## Current state
- **Version:** 0.22.0
- **Open issues:** 11 (#48-49, #57, #63-70)
- **Resolved this session:** #58 (stale), #59 (PR #62), #60 (decomposed), #61 (PR #71)
- **Remaining blockers:** `pgapi-partial-api-coverage` (decomposed into #63-70), `cmd-propagate-bypasses-api` (fixed but belief not yet retracted in expert repo)
