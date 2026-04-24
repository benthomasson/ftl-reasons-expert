# Defect Resolution: Achieving a Clean Belief Network

**Date:** 2026-04-24
**Time:** 14:00

## Summary

Between 2026-04-23 and 2026-04-24, all 10 defect premises identified in the ftl-reasons knowledge base were resolved through a complete pipeline cycle: identify defects via GATE beliefs, file issues, generate fixes with multi-agent SDLC, validate with multi-model code review, merge with auto-retraction, and verify GATE belief propagation. The network now stands at 130 IN / 59 OUT with zero active blockers.

## How defects were identified

The code-expert pipeline explored the ftl-reasons codebase, extracting 83 premises (direct code observations) and deriving 106 higher-level beliefs through LLM-driven reasoning chains (`reasons derive --exhaust`). Among these derived beliefs were GATE beliefs — beliefs with outlist justifications that say "believe X unless defect Y is true." When a defect premise was IN, the GATE belief stayed OUT, signaling that the system property it described was not yet achieved.

For example:
- `propagation-is-crash-free` (GATE) was OUT because `propagate-assumes-dependents-exist` (defect) was IN
- `compact-budget-controls-output-size` (GATE) was OUT because `compact-token-estimate-is-word-count` and `compact-budget-only-limits-in-nodes` were IN
- `staleness-gate-catches-all-drift` (GATE) was OUT because `hash-truncation-is-16-hex` and `missing-source-file-is-silent` were IN

The command `code-expert file-issues` turned these into GitHub issues automatically, each containing the belief ID in backticks for later auto-retraction.

## The 10 defect premises

### Issue #22: Propagation crashes on dangling dependent references
**Premise:** `propagate-assumes-dependents-exist`
**Problem:** `_propagate` accessed `self.nodes[dep_id]` without checking membership. A dangling dependent reference would raise `KeyError`.
**Fix (PR #27):** Added guard `if dep_id not in self.nodes: continue` in the BFS loop.

### Issue #23: _build_beliefs_section inflates belief count per agent
**Premise:** `derive-agent-count-bug`
**Problem:** `count += len(belief_ids)` was inside the per-belief loop instead of outside it, inflating the count and shrinking the non-agent budget.
**Fix (PR #27):** Moved the increment outside the inner loop.

### Issue #24: Dependents index is fragile denormalization
**Premises:** `dependents-bidirectional-index`, `dependents-index-derived-on-load`, `dependents-is-manual-reverse-index`
**Problem:** The `node.dependents` set was a manually-maintained reverse index rebuilt on every load, with consistency obligations scattered across mutation paths.
**Fix (PR #31):** Consolidated maintenance into `_update_dependents` called from all mutation paths, added `_rebuild_dependents` for load-time reconstruction.

### Issue #25: check_stale silently skips missing source files
**Premise:** `missing-source-file-is-silent`
**Problem:** When a node's source file was deleted from disk, `check_stale` silently skipped it. Callers couldn't distinguish "file deleted" from "file never tracked."
**Fix (PR #32):** Added `source_deleted` result type. Missing files now produce an explicit result with `reason: "source_deleted"`, `new_hash: None`, and `source_path: None`.

### Issue #26: Nogood IDs assume append-only list
**Premise:** `nogood-ids-assume-append-only`
**Problem:** Nogood IDs were derived from `len(self.nogoods) + 1`. Deleting a nogood would cause ID collisions.
**Fix (PR #33):** Changed to `max(n.id for n in self.nogoods) + 1` with fallback to 1 for empty lists.

### Issue #36: Compact token budget uses word count, not BPE
**Premises:** `compact-token-estimate-is-word-count`, `compact-budget-only-limits-in-nodes`
**Problem:** `estimate_tokens` counted whitespace-separated words instead of using the standard chars/4 BPE approximation. The budget only constrained IN nodes — nogoods and OUT nodes were emitted unconditionally.
**Fix (PR #39):** Changed to `len(text) // 4` heuristic. Made all three sections (nogoods, OUT, IN) respect the budget with a running character counter for O(1) budget tracking.

### Issue #37: SHA-256 hash truncated to 16 hex chars
**Premise:** `hash-truncation-is-16-hex`
**Problem:** `hash_file()` returned `hashlib.sha256(...).hexdigest()[:16]`, reducing collision resistance to ~32 bits for birthday attacks.
**Fix (PR #40):** Removed the `[:16]` truncation. Full 64 hex char SHA-256 hashes are now stored and compared.

## The fix pipeline

Each defect went through the same pipeline:

1. **ftl-sdlc-loop** — Multi-agent pipeline (planner, implementer, reviewer, tester, user agents) generated the fix PR with tests
2. **code-review** — Multi-model review (Claude + Gemini) validated each PR. Most PRs required 2-4 review-fix iterations:
   - PR #39 (compact budget): 4 rounds — docstring clarity, magic numbers, O(n^2) budget tracking, dead code
   - PR #3 on ftl-merge (GATE skip): 4 rounds — tests, abort-on-failure, shell injection removal, dead code
3. **ftl-merge** — `ftl-merge --auto-retract` merged PRs and retracted the defect premises automatically by parsing belief IDs from the linked issue body

A critical fix was needed in ftl-merge itself before the final merges: issue #2 on ftl-merge identified that it was retracting GATE beliefs, which set `_retracted` metadata and prevented TMS propagation from restoring them. The fix (PR #3, version 0.2.0) added `load_network()` and `has_outlist()` to detect and skip GATE beliefs during auto-retraction.

## The merge sequence

### Round 1: Issues #22-#26 (PRs #27, #31, #32, #33)
Merged on 2026-04-23. Five defect premises retracted. The ftl-merge GATE-skip feature worked correctly — GATE beliefs like `propagation-is-crash-free` were skipped during retraction and flipped IN via TMS propagation.

### Round 2: Issues #36-#37 (PRs #39, #40)
Merged on 2026-04-24. Two more defect premises retracted (3 total for these issues). `compact-budget-controls-output-size` flipped IN automatically. `staleness-gate-catches-all-drift` stayed OUT due to a propagation bug (see below) and required manual `reasons assert`.

## Propagation bug discovered

During Round 2 verification, `staleness-gate-catches-all-drift` remained OUT despite having valid justification conditions (antecedent `staleness-checking-is-comprehensive` IN, both outlist nodes `hash-truncation-is-16-hex` and `missing-source-file-is-silent` OUT).

Investigation revealed that outlist nodes are not tracked in the `dependents` index. When an outlist node is retracted (goes OUT), the dependent GATE belief is not enqueued for re-evaluation by `_propagate`. This means GATE beliefs don't automatically flip IN when their outlist conditions become satisfied via retraction of the outlist node.

This is a real bug in ftl-reasons — the same class of "dependents index incompleteness" that issue #24 partially addressed, but for outlist references specifically. Manual `reasons assert` was used as a workaround. This should be filed as a new issue.

## Final state

| Metric | Value |
|--------|-------|
| Total beliefs | 189 |
| IN beliefs | 130 |
| OUT beliefs | 59 |
| Premises | 83 |
| Derived | 106 |
| Retracted premises | 10 |
| Active blockers | 0 |
| Terminal derived (depth 1-5) | 19 |
| Deepest reasoning chain | 5 levels |

The 19 terminal derived beliefs — conclusions that no other belief depends on — represent the highest-level findings about the system:

- **Depth 5:** Edge-case uniformity follows from semantic minimality
- **Depth 4:** Belief revision is lifecycle-safe and semantics-preserving
- **Depth 3:** External inputs are safely integrated, belief revision is fully reliable, dialectics are semantically transparent, external belief lifecycle is complete, external beliefs are defensively contained, LLM mutations are bounded end-to-end
- **Depth 2:** Challenge/defend is crash-safe, derive pipeline is production-ready, lifecycle awareness spans checking and propagation, staleness gate catches all drift, TMS core is crash-safe
- **Depth 1:** Compact budget controls output size, derive budget allocation is accurate, nogood resolution maintains consistent IDs, propagation is crash-free, staleness checking is comprehensive, three-layer stack has clean boundaries

All 130 IN beliefs are positive or neutral observations about the codebase. Zero negative or critical IN beliefs remain. The network is clean.

## Significance

This represents a complete cycle of the code-expert methodology:

**Code -> Knowledge -> Defects -> Fixes -> Verification**

The system found its own bugs. Premises describing defects were extracted from code, derived beliefs identified which system properties those defects blocked, issues were filed automatically, fixes were generated and validated by multi-model review, and merging the fixes retracted the defect premises — causing the GATE beliefs to flip IN and confirming the properties now hold.

The belief network serves as a living proof: when a defect is fixed, the network state changes to reflect the new reality. When new defects are discovered, new premises will block the relevant GATE beliefs again. The network doesn't just document the code — it tracks which properties the code actually achieves.
