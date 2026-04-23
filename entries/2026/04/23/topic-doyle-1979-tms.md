# Topic: Doyle's original Truth Maintenance System paper; understanding SL/CP justification semantics and the non-monotonic reasoning model this code implements

**Date:** 2026-04-23
**Time:** 16:49

Now I have a complete picture. Here's the explanation:

---

# Doyle's Truth Maintenance System in `ftl-reasons`

## The Paper: What Problem Does It Solve?

Jon Doyle's 1979 paper introduced the **Truth Maintenance System (TMS)** to solve a fundamental problem in AI reasoning: how do you keep a set of beliefs consistent when the world changes? Classical logic is *monotonic* — once you derive a conclusion, it stays true forever. But real reasoning is *non-monotonic*: you believe things provisionally, retract them when counter-evidence arrives, and restore them when the counter-evidence is itself defeated.

The core insight is to separate the **reasoning** (which conclusions follow from which premises) from the **bookkeeping** (which conclusions are currently believed given which premises are currently held). The TMS handles the bookkeeping automatically.

## The Data Model

The system is built on three data structures defined in `reasons_lib/__init__.py`:

**Node** (line 26): A belief with an `id`, `text`, `truth_value` (`"IN"` or `"OUT"`), and a list of `justifications`. It also maintains a `dependents` set — a reverse index of nodes whose truth depends on this one. A node with no justifications is a **premise** (IN by default). A node with justifications is **derived** (IN only if at least one justification is valid).

**Justification** (line 7): A reason for believing a node. Has a `type` (`"SL"` or `"CP"`), an `antecedents` list (the *inlist* — nodes that must be IN), and an `outlist` (nodes that must be OUT). This dual-list structure is the key to non-monotonic reasoning.

**Nogood** (line 44): A recorded contradiction — a set of node IDs that cannot all be IN simultaneously.

## SL Justification Semantics

SL stands for **Support List** — Doyle's term for the most common justification type. The semantics are implemented in `network.py:717` (`_justification_valid`):

```python
inlist_ok = all(a in self.nodes and self.nodes[a].truth_value == "IN"
                for a in j.antecedents)
outlist_ok = all(o not in self.nodes or self.nodes[o].truth_value == "OUT"
                for o in j.outlist)
return inlist_ok and outlist_ok
```

An SL justification is valid when:
1. **All antecedents (inlist) are IN** — every supporting premise is currently believed
2. **All outlist nodes are OUT** — no defeating evidence is currently believed

The truth of a *node* is then computed in `_compute_truth` (line 704): a node is IN if **any** of its justifications is valid. This disjunction over justifications means a node can survive having one justification defeated, as long as it has an alternative. The test at `tests/test_outlist.py:142` demonstrates this: node X has two justifications, one gated by outlist and one unconditional. Defeating the first doesn't kill X because the second holds.

The **outlist** is where non-monotonicity enters. Without it, SL would just be monotonic forward chaining. The outlist lets you express defaults: "believe X *unless* Y is believed" (`Justification(type="SL", antecedents=[], outlist=["y"])`). When Y arrives, X retracts automatically; when Y is itself retracted, X comes back. This is the formal mechanism behind the `challenge`/`defend` system (lines 374–493) and the `supersede` operation (line 287).

## CP Justification Semantics

CP stands for **Conditional Proof** — justification based on the *consistency* of a set of assumptions rather than their truth. In Doyle's paper, CP justifications allow deriving conclusions from hypothetical assumptions that haven't been contradicted. In this implementation, CP is treated identically to SL at the evaluation level (line 723: `if j.type in ("SL", "CP")`). The distinction exists in the data model for semantic clarity: SL says "this holds *because* these things are true," while CP says "this holds *assuming* these things are consistent." The API exposes them as separate `--sl` and `--cp` flags (`api.py:131-136`).

## Propagation: The Core Algorithm

When any node's truth value changes, the change must ripple through the dependency graph. This is implemented as BFS in `_propagate` (line 675):

1. Starting from the changed node, visit all dependents
2. For each dependent, recompute its truth value via `_compute_truth`
3. If the value changed, record it and add *its* dependents to the queue
4. Skip explicitly retracted nodes (`_retracted` metadata flag)

This handles both **retraction cascades** (a premise goes OUT → all nodes that depended on it recompute, potentially going OUT → their dependents recompute...) and **restoration** (a premise comes back IN → dependents may become valid again → *their* dependents may restore...).

The `dependents` set on each node is the reverse index that makes this efficient. It's built during `add_node` (lines 61-65) — both inlist *and* outlist nodes register as potential sources of change, because a change to either can affect the dependent's truth value.

## Dependency-Directed Backtracking

When a contradiction is detected (a set of nodes that shouldn't all be IN), the system doesn't just retract an arbitrary node. `add_nogood` (line 244) uses `find_culprits` (line 197) to:

1. **Trace backward** through justification chains via `trace_assumptions` (line 127) to find the *premises* supporting each contradicted node
2. **Score premises by entrenchment** (`_entrenchment`, line 162) — premises backed by source files, with many dependents, or typed as AXIOM/OBSERVATION score higher
3. **Retract the least entrenched premise** — speculative assumptions are retracted before hard evidence

This preserves the maximum amount of well-supported knowledge while resolving the contradiction with minimal disruption.

## The Non-Monotonic Reasoning Model

Put together, the system implements a specific form of non-monotonic reasoning:

- **Default reasoning**: "Believe X unless there's a reason not to" — implemented via outlist-only justifications (`Justification(type="SL", antecedents=[], outlist=["counter-evidence"])`)
- **Defeasible inference**: "X follows from A, but can be overridden by Y" — implemented via `Justification(type="SL", antecedents=["a"], outlist=["y"])`
- **Dialectical argumentation**: Challenges defeat beliefs; defenses defeat challenges. Both use the same outlist mechanism (`challenge` at line 374 adds the challenge node to the target's outlist; `defend` at line 449 adds a defense to the *challenge's* outlist, neutralizing it)
- **Belief revision**: When evidence changes, the entire network updates automatically — no manual bookkeeping

The persistence layer (`storage.py`) wraps all this in SQLite with WAL journaling, so propagation cascades are ACID transactions. The `_with_network` context manager in `api.py:33` ensures load-operate-save atomicity.

---

## Topics to Explore

- [file] `tests/test_backtracking.py` — Full test suite for dependency-directed backtracking, including entrenchment scoring, cascade-from-retracted-premise, and the "retract speculation not evidence" invariant
- [function] `reasons_lib/network.py:challenge` — The dialectical argumentation mechanism: how challenges and defenses compose through nested outlist relationships
- [function] `reasons_lib/network.py:supersede` — How the outlist enables reversible supersession of beliefs — a pattern that reuses the same propagation machinery for version control of ideas
- [file] `reasons_lib/derive.py` — Automated derivation of higher-order beliefs by combining existing conclusions, including cross-agent reasoning chains and GATE (outlist-gated) beliefs
- [general] `namespace-agent-model` — How agent namespaces (e.g., `agent-name:belief-id`) partition the belief space, with a single `ns:active` premise that can cascade-retract an entire agent's contributions

## Beliefs

- `sl-justification-is-conjunction-of-inlist-disjunction-of-outlist` — An SL justification is valid iff ALL antecedents are IN AND ALL outlist nodes are OUT; a node is IN iff ANY of its justifications is valid (conjunction within, disjunction across)
- `cp-and-sl-evaluated-identically` — CP and SL justifications use the same validity check in `_justification_valid` (line 723); the distinction is semantic, not computational
- `propagation-is-bfs-not-dfs` — Truth value propagation uses breadth-first search through the dependents graph, ensuring level-by-level consistency updates
- `retracted-nodes-stay-in-network` — Retracted nodes are marked OUT with `_retracted` metadata but remain in the network, enabling restoration without rederivation
- `backtracking-retracts-least-entrenched-premise` — When resolving a nogood, `find_culprits` sorts by entrenchment score (lowest first) and retracts the least entrenched premise, not the contradiction nodes themselves

