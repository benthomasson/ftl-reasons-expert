---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Investigate Doyle's original algorithm and the assumption that `N+1` passes suffice; test what happens with justification cycles and retracted-node chains.

# Doyle's Algorithm, `N+1` Passes, and the Cycle/Retraction Edge Cases

## What Doyle's algorithm actually does

Doyle's 1979 TMS maintains a network of *nodes* (beliefs), each with a `truth_value` of `IN`/`OUT`. A node is `IN` iff it has at least one **well-founded** justification whose antecedents are all `IN` and whose outlist is all `OUT`. When a premise is retracted or restored, that bit changes and the change must cascade to all downstream nodes until a **fixpoint** is reached — no node's status can change under the rule "IN iff a valid justification makes it IN."

The codebase references this repeatedly:

- `reasons/__init__.py:1` — "data model based on Doyle (1979)"
- `reasons/network.py:3` — "Implements Doyle's (1979) TMS algorithm"
- `CLAUDE.md:95` / `reasons/__init__.py:49` — retracted nodes stay in the DB (`retracted_at` field) so they can be restored without rederivation.

## Where `N+1` shows up

The `N+1` bound is the fixpoint guard. It appears in **seven** propagation loops:

| File | Line | Code |
|---|---|---|
| `reasons/network.py` | 911 | `max_iterations = len(self.nodes) + 1` |
| `reasons/pg.py` | 1601 | `max_iterations = cur.fetchone()[0] + 1` (node count query) |
| `reasons/api.py` | 2137 | `max_passes = len(remaining) + 1` |
| `reasons/import_agent.py` | 197 | `max_passes = len(remaining) + 1` |
| `reasons/import_beliefs.py` | 202 | `max_passes = len(remaining) + 1` |
| `reasons/pg.py` | 2267, 2415 | `max_passes = len(remaining) + 1` |

The comment at `reasons/network.py:907` explains the intent: *"Iterates to a fixpoint so cascading changes propagate regardless of [order]."* All the import paths (agent, beliefs, PG variants) use the same trick: repeatedly scan the set of not-yet-settled nodes, and stop after `N` nodes have appeared plus one final "no change" pass.

## Why `N+1` is *thought* to be sufficient

The intuition is graph-theoretic:

- In the **acyclic** case, the longest justification chain has at most `N-1` edges (where `N = |nodes|`). A cascade from the top of that chain to the bottom needs at most `N-1` hops, so `N-1` productive passes + 1 verification pass = `N`.
- The `+1` is a redundant "look again — did anything change?" pass, required to correctly *terminate* rather than to *compute*.

The timings are visible in `tests/test_dialectical.py` (`TestDialecticalChain`), which builds actual chains:

- `test_three_level_chain` (line ~204) creates: premise `a` → `challenge-a` (outlist on `a`) → `defense-challenge-a` (outlist on `challenge-a`) → `challenge-defense-challenge-a` (outlist on `defense-challenge-a`). That's a **4-hop dependency chain**, and the assertion `net.nodes["a"].truth_value == "IN"` at the end confirms the fixpoint loop walked all 4 levels.
- `test_challenge_defense` (line ~181) is a **3-hop** chain, also covered.

So the `N+1` bound tested indirectly: a 5-node chain (`a`, challenge, defense, counter-challenge, counter-defense) needs ≥4 passes to fully propagate. With `N+1` where `N` is the full node count (~8 in that test), it comfortably fits.

The `+1` also matters when the "productive" pass count is at the theoretical max — without the extra pass, the loop would exit after making a change on the last iteration without verifying that the change itself cascaded further.

## Where `N+1` breaks down: justification cycles

The fixpoint bound is **only safe for DAGs**. Doyle's system *permits* cycles in the justification graph (an SL cycle A←B, B←A never grounds out), and the correct semantic is: nodes in a cycle with no external support are `OUT` ("no well-founded support"). A naive `for _ in range(len(nodes)+1)` loop **does not detect this distinction** — it just terminates with whatever truth values the last pass produced. That's arguably correct (acyclic + cyclic regions both reach a stable assignment within `N` passes because the number of possible states is bounded), but it's worth verifying.

Two things in the observations are relevant:

1. **`reasons/derive.py:118-121`** — `_get_depth` has an explicit cycle guard:
   ```python
   memo[node_id] = 0  # cycle guard
   ```
   This proves the codebase *knows* cycles exist in the graph (they even leak into the derive prompt via `_get_depth`), but the guard lives in the *derivation depth* helper, not in the fixpoint loop.

2. **No test named "cycle" or "cyclic" appears** in `tests/test_backtracking.py`, `tests/test_dialectical.py`, or `tests/test_derive.py`. The grep for `cycle|cyclic` returns retraction matches, not cycles. **There is no observable test of a justification cycle in the gathered data** — this is a coverage gap you should confirm.

If a cycle is present and the fixpoint loop naively re-evaluates each iteration, then `N+1` iterations is still a *sufficient* bound (each node can only flip a bounded number of times before stabilizing), but the loop **cannot distinguish "stable fixpoint" from "still oscilating when we ran out of passes."** The `max_iterations` cap silently returns whatever state the last pass computed. Any cycle therefore needs either:
- a final "no-change" pass that succeeds (loop exits early), or
- a semantic guarantee that cycles can't be constructed (which the `challenge`/`defend` API in `tests/test_dialectical.py` does *not* enforce — nothing prevents `net.challenge("challenge-a", ...)` from being called *after* `net.challenge("a", ...)` was already made for the same target).

## Where `N+1` breaks down: retracted-node chains

Doyle's insight (recorded at `CLAUDE.md:95` and `reasons/__init__.py:49`) is that a retracted node stays in the DB. This is essential for **restoration**: when you undo the retraction, downstream nodes that are still justified flip back `IN` without rederivation. The tests exercise this:

- `test_dismiss_challenge_restores_target` (`tests/test_dialectical.py`) — retract `challenge-a`, and `a` comes back `IN`.
- `test_retract_defense_reinstates_challenge` — retract a defense node → defense-challenge goes OUT → challenge comes back IN → target goes OUT.
- `TestBacktrackingInNogood.test_cascade_from_retracted_premise` (`tests/test_backtracking.py`) — retracts a premise `d`, asserts `c` and `b` stay IN because their justification chain (`a → b → c`) is untouched.

The **risky case** is a *chain* of retractions/restorations where the fixpoint loop interacts with `what_if_*` API methods. `reasons/api.py:512` (`what_if_retract`) explicitly does:
> "Loads the network read-only, performs the retraction in memory, [does not save]"

If the in-memory retraction triggers a fixpoint cascade but the *restoration* would too, calling `what_if_retract` followed by `what_if_restore` (line 588) exercises a full retract-then-restore cycle. The observations show these are tested (they appear at `reasons/api.py:512, 588`), but the *in-memory fixpoint* reuses the same `N+1` bound from `network.py:911`. Because `what_if_retract` is only a single operation (no chain of restorations), the `N+1` bound should hold — but there's no direct test that a **restoration chain** (retract a defence → challenge comes back → third-party node depending on that challenge cascades) stays within bounds.

## Gaps in the observations

The gathered code does **not** show the actual body of the propagation loop:

- `reasons/network.py` lines **900–950** are only excerpted (we see the header/comment at 907 and the loop guard at 911–912, but not what happens inside).
- `reasons/pg.py` lines **1593–1620** — we see the header of `recompute_all`-style function at 1593 and the cap at 1601–1603, not the per-node logic.
- We don't see whether the loop uses a **dirty set** (only re-evaluate nodes whose antecedents changed on the last pass) or a **full sweep** (re-evaluate all nodes). A full sweep means `N+1` iterations of O(N) work = O(N²); a dirty set means O(N) total. That affects whether `N+1` is even the right bound for performance, not just correctness.

Because of these gaps, we cannot confirm from the observations alone whether cycles and restorations are *handled* or merely *tolerated*. To answer the question definitively you need to read those specific line ranges.

## Test status summary

| Scenario | Tested? | Where |
|---|---|---|
| Linear chain (claim → 4 levels deep) | ✅ | `tests/test_dialectical.py::TestDialecticalChain.test_three_level_chain` |
| Retract → cascade → restore | ✅ | `tests/test_dialectical.py::test_dismiss_challenge_restores_target`, `test_retract_defense_reinstates_challenge` |
| Backtracking victim selection | ✅ | `tests/test_backtracking.py::TestBacktrackingInNogood` |
| Long justification chain within `N+1` budget | ⚠️ implicit only | Same as above — no test with N linearly-chained derived nodes |
| Justification cycle (A↔B) | ❌ | No observable test |
| Retract-restore-retract cycle with propagation | ❌ | No observable test |
| Fixpoint loop's handling of `max_iterations` exhaustion | ❌ | No test asserts that "still changing after N+1" raises or logs |

---

## Topics to Explore

- [function] `reasons/network.py:propagate` — The actual fixpoint loop body between lines 907–950; whether it uses a dirty set or full sweep, and whether it distinguishes "converged" from "exhausted iterations."
- [function] `reasons/pg.py:1601` — The PG backend's `recompute_all` (or equivalent) — does the SQL implementation handle cycles identically to the in-memory one? Divergence here is a likely bug source.
- [general] `justification-cycle-handling` — Construct an explicit SL cycle via `net.add_node(..., justifications=[Justification(...)])` and observe whether the fixpoint produces `OUT`/`OUT` (Doyle-correct) or oscillates to `IN`/`OUT` depending on iteration order.
- [file] `tests/test_network.py` — It calls out "Retraction cascades — Doyle's core mechanism" (line 54); check whether any test exceeds `N` propagation hops or constructs cycles.
- [general] `restore-chain-bounds` — Build a chain of retractions and restorations and verify the `max_passes = len(remaining) + 1` heuristic in `reasons/api.py:2137` never terminates early on a still-changing state.

## Beliefs

- `n-plus-one-is-node-count-bound` — The `max_iterations = len(nodes) + 1` guard in `reasons/network.py:911` and `reasons/pg.py:1601` is sized by total node count, not by the longest justification chain.
- `fixpoint-bound-is-implicit-not-verified` — Neither the in-memory nor the PG loop appears to detect when `max_iterations` is exhausted while nodes are still flipping; correctness relies on the `N+1` bound being sufficient rather than on an explicit convergence check.
- `retracted-nodes-are-preserved-for-restoration` — Nodes have a `retracted_at` field (`reasons/__init__.py:49`) and are kept in the DB after retraction so restoration does not re-derive them, matching Doyle's 1979 formulation (`CLAUDE.md:95`).
- `dialectical-chains-use-outlist-justifications` — The `challenge`/`defend` API in `tests/test_dialectical.py` implements Doyle §6 argumentation as outlist edges: `challenge-a` going IN forces `a` OUT, and `defense-challenge-a` going IN forces `challenge-a` OUT.
- `cycle-guard-exists-in-depth-not-propagation` — `_get_depth` in `reasons/derive.py:121` explicitly guards against cycles, but no corresponding guard is visible in the propagation loop's `max_iterations` bound.
