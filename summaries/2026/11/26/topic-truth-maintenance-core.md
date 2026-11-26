---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The Doyle model (IN/OUT, SL vs CP justifications, cascade, restoration) as the conceptual spine every file serves.

# The Doyle Model as the Conceptual Spine

Every file in this repository is a facet of one algorithm: Doyle's 1979 Truth Maintenance System (aka Reason Maintenance System). CLAUDE.md opens with the lineage:

> "Reasons — a belief tracking system based on Doyle's 1979 Truth Maintenance System."

README.md line 3 makes the same claim, and the SKILL.md file exists purely to teach LLM operators how to speak the model's vocabulary. Below is how the four pillars — IN/OUT, SL/CP, cascade, restoration — thread through every file.

## 1. IN / OUT: the two-valued truth state

The entire system collapses all beliefs to a binary.

- `reasons/__init__.py:35` — `truth_value: str = "IN"  # IN or OUT`
- `reasons/__init__.py:30-31` — "A node is IN if ANY of its justifications is valid. A premise has no justifications and is IN by default."
- `reasons/storage.py` SCHEMA — `truth_value TEXT NOT NULL DEFAULT 'IN'` persisted in the `nodes` table; propagated state survives restarts.
- `.claude/skills/reasons/SKILL.md:82` — status shows `[+]` for IN, `[-]` for OUT with an `IN/total` tally.

Crucially, **there is no STALE status**. CLAUDE.md is explicit:

> "No STALE status — uses IN/OUT only; staleness triggers retraction"

That means `reasons/verify.py` (which computes CONFIRMED / STALE / PARTIAL / INCONCLUSIVE) is a *feeder* to the truth value, not a third state — a STALE verdict becomes input to a retraction decision, not a stored enum.

## 2. SL: the Support List justification (monotonic)

The primary justification type.

- `reasons/__init__.py:11-12` — "SL (Support List): node is IN iff ALL antecedents (inlist) are IN AND ALL outlist nodes are OUT"
- `reasons/api.py:300-302` — the CLI's `--sl a,b` and `--unless y` flags map directly to `antecedents` and `outlist`.
- `reasons/api.py:326-328` — two dispatch branches: `any_mode=True` expands `--sl a,b` into one justification per antecedent (OR semantics); default yields one multi-antecedent justification (AND semantics).
- `reasons/__init__.py:30` — **"A node is IN if ANY of its justifications is valid."** Multiple SL justifications => disjunction of conjunctions. This is what the README calls "multiple justifications" and it's why a node only goes OUT when *every* path breaks.

## 3. CP: Conditional Proof justification

The less-used but architecturally distinct type.

- `reasons/__init__.py:13` — "CP (Conditional Proof): node is IN iff assumptions are consistent"
- `reasons/api.py:301, 331` — `--cp a,b` creates `Justification(type="CP", ...)` with the same list shape as SL but different semantics: the antecedents are *assumptions under which the node is provable*, so the node's truth depends on the assumptions' **consistency**, not their IN-status.

The schema (`reasons/storage.py`, `justifications` table) stores `type` as a plain TEXT column and antecedents/outlist as JSON, so SL and CP share a storage path but diverge in the propagation predicate inside `reasons/network.py`.

## 4. Cascade: automatic retraction propagation

The crown jewel of the model — the reason this is not just a graph database.

- `.claude/skills/reasons/SKILL.md:35` — "Retraction cascade: When a node goes OUT, all dependents whose justifications become invalid also go OUT — automatically."
- README example: `retract source-uses-langgraph` prints `Retracted: source-uses-langgraph, topology-is-static, no-runtime-modification` — three nodes collapse from one command.
- CLAUDE.md design decision: "BFS propagation ensures all dependents are recomputed when a node changes."
- `reasons/storage.py:1-6` module docstring: "ACID transactions ensure propagation cascades are atomic." If the process dies mid-cascade, the network cannot be left in an inconsistent half-retracted state.
- `tests/test_backtracking.py:test_cascade_from_retracted_premise` (lines around "A → B → C. Nogood(C, D)") verifies the cascade reaches transitively through justification chains.

The cascade is *directed* — it flows forward from a changed node to dependents. Backward reasoning is handled separately by `trace_assumptions` (`tests/test_backtracking.py::TestTraceAssumptions`), which walks the inlist from a conclusion back to root premises.

## 5. Restoration: retraction is reversible

The other half of Doyle's insight.

- `.claude/skills/reasons/SKILL.md:36` — "Restoration: When a retracted node comes back IN, dependents are recomputed — no manual rederivation needed."
- CLAUDE.md design decision: **"Retracted nodes stay in DB (Doyle's insight: enables restoration without rederivation)."**
- README example: after `assert graph-has-cycles`, `Asserted: source-uses-langgraph, topology-is-static, no-runtime-modification` — the *entire* downstream chain comes back from one command.
- This is why `reasons/storage.py:save` rewrites justifications and nodes wholesale rather than deleting OUT nodes — an OUT node is a *dormant* belief, not an erased one.
- `reasons/nogood` (`reasons/__init__.py:54`) — "these node IDs cannot all be IN simultaneously" — depends on retracted nodes still being present so backtracking can *choose which* premise to retract via `find_culprits` rather than arbitrarily deleting.

## 6. The non-monotonic extensions (outlist, nogood, challenge)

Doyle's original model was monotonic. This codebase extends it while keeping the IN/OUT skeleton:

- **Outlist** (`.claude/skills/reasons/SKILL.md:37`; `reasons/__init__.py:15-17`) — "believe X unless Y" — `unless` param in `reasons/api.py:302`. A node can be OUT a premise holds, and IN if its blocker is OUT. This is what lets `default-approx` survive until `strong-field` is asserted (README end-to-end example).
- **Nogood / dependency-directed backtracking** (README "Core Ideas", `tests/test_backtracking.py`): when a contradicting set is recorded, `net.add_nogood` walks `trace_assumptions` to find premises and uses `_entrenchment` (see `test_evidence_premise_more_entrenched_than_derived`) to pick the *cheapest* premise to retract — evidence survives, speculation dies.
- **Dialectical argumentation** (`.claude/skills/reasons/SKILL.md:39`; `import reasons` in `reasons/__init__.py`): `challenge` builds a node that is IN and is added to the target's outlist, forcing the target OUT. `defend` neutralizes the challenge, causing restoration. This is outlist semantics applied socially — the same mechanism, different vocabulary.

## 7. How every file serves the spine

| File | Role in the Doyle model |
|---|---|
| `reasons/__init__.py` | Data model — `Node`, `Justification(type=SL/CP)`, `Nogood` (lines 11-54) |
| `reasons/network.py` | The propagator — BFS recompute; `trace_assumptions`, `find_culprits`, `_entrenchment` |
| `reasons/storage.py` | Durability of IN/OUT + justifications + log, made ACID for cascade atomicity (lines 1-6, SCHEMA) |
| `reasons/api.py` | CLI/Python surface that maps `--sl`/`--cp`/`--unless` onto Justification rows (lines 300-334) |
| `reasons/cli.py` | The verbs: add / retract / assert / status / explain / trace / challenge / defend |
| `reasons/derive.py` | LLM prompt (`DERIVE_PROMPT`, lines ~20-80) that **retells** the model to a language model so it proposes new SL and GATE nodes that fit the graph |
| `reasons/verify.py` | Fresh evidence check that *feeds* retraction — never writes STALE, only verdicts a caller can act on |
| `reasons/import_beliefs.py` | Migrates legacy beliefs into SL nodes and retraction (`.claude/skills/reasons/SKILL.md:122-124`) |
| `reasons/export_markdown.py` | Round-trip: IN → SL-justified, no-dependency IN → premise, OUT → retracted |
| `reasons/check_stale.py` | Source-hash comparison — "this belief's evidence has changed; consider retracting" |
| `reasons/merkle.py` | Per-justification content hashes, integrity check on the justification store |
| `reasons/compact.py` | Compresses the IN/total state to a token budget without breaking the model's semantics |
| `tests/test_backtracking.py` | Verifies `find_culprits` retracts *premises*, not derived nodes — Doyle's correction to naive TMS |
| `.claude/skills/reasons/SKILL.md` | Instructional layer — tells LLM operators that "retract / assert is the most important operation" |
| `CLAUDE.md` | The rationale document — records *why* the model was chosen and where the extensions diverge |

Every command in the README command table (`reasons retract`, `assert`, `propagate`, `explain`, `trace`, `nogood`, `challenge`, `defend`) is a window onto the same underlying IN/OUT graph with SL/CP justifications on top.

## What the observations don't show

The gathered snippets don't include `reasons/network.py`'s propagation loop, `parse_sl`/`parse_cp` predicates, or the exact CP-consistency check. The *contract* is documented (`__init__.py:11-13`) and tested (`tests/test_outlist.py`, `tests/test_backtracking.py`), but the concrete predicate is only partially visible — you'd want to open `reasons/network.py` to see how `_recompute` evaluates SL vs CP against current IN/OUT values.

## Topics to Explore

- [function] `reasons/network.py:propagate` — The BFS engine that actually walks dependents when a truth value flips; the code realizing the "cascade" described in docstrings.
- [function] `reasons/network.py:find_culprits` — Backward search that identifies which premises to retract; the exact counterpart to `trace_assumptions` used by `add_nogood`.
- [file] `tests/test_outlist.py` — The test file pinning down non-monotonic ("unless") semantics — the boundary between Doyle's monotonic TMS and this implementation.
- [function] `reasons/api.py:add_node` (near line 300) — The single dispatch point that translates `--sl` / `--cp` / `--unless` / `any_mode` into Justification rows; the fork between SL and CP.
- [file] `tests/test_dialectical.py` — How `challenge`/`defend` are built entirely on top of outlist semantics rather than as a separate subsystem.

## Beliefs

- `reasons-implements-doyle-1979-tms` — README.md line 3 and CLAUDE.md both state Reasons is an implementation of Doyle's 1979 Truth Maintenance System.
- `truth-value-is-restricted-to-in-or-out` — No STALE state exists; staleness in `verify.py` triggers retraction rather than becoming a stored status (CLAUDE.md design decisions).
- `retracted-nodes-remain-in-storage-enabling-restoration` — `Storage.save` rewrites OUT nodes rather than deleting them, so `assert` can restore dependents without rederivation, matching Doyle's insight recorded in CLAUDE.md.
- `sl-justification-requires-all-inlist-in-and-all-outlist-out` — Stated verbatim in `reasons/__init__.py:11-12`; enforced by the propagation predicate and exercised throughout `tests/test_backtracking.py`.
- `node-with-multiple-justifications-is-in-if-any-is-valid` — `reasons/__init__.py:30` — disjunction of justifications, so a node goes OUT only when every justification is invalidated.
- `nogood-backtracking-retracts-premises-not-derived-nodes` — `tests/test_backtracking.py::TestBacktrackingInNogood::test_retracts_premise_not_derived` asserts a premise (not the derived node) is chosen as the culprit, using `_entrenchment` scoring.
