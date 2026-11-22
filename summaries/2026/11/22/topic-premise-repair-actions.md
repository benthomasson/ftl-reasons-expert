---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Domain semantics of `rewrite` versus `retract` in the Truth Maintenance System.

## Domain semantics of `rewrite` versus `retract`

Short version: **`retract` is a first-class TMS truth-value operation; `rewrite` is not a TMS operation at all** — it appears in two different domains, only one of which is about the belief network's truth values, and neither is a truth-value flip. The relationship between them is best seen in the deduplication path, where they are *complementary*, and in the premise-repair path, where they are *alternatives*.

### `retract` — a truth-value transition on the network

- The public entry point is `retract_node(node_id, reason, db_path, ...)` at `reasons/api.py:473`, which delegates down to `net.retract(node_id, reason=reason)` at `reasons/api.py:489`. The MCP server exposes it as a tool at `reasons/mcp_server.py:146`.
- It is **cascading and network-wide**: "retracting it cascades OUT" (`reasons/api.py:221`), and the skill docs call it "the most important operation" because the network recomputes dependents automatically (`.claude/skills/reasons/SKILL.md:73-76`).
- It is **stateful and reversible**: retraction records `retracted_at` on the node (`reasons/__init__.py:49`, persisted by `reasons/storage.py`), sets a `_retracted` metadata flag (CHANGELOG.md:34), and the node can be restored with `assert` (SKILL.md:79). Retract is a *flip to OUT*, not a delete.
- It leaves a **tombstone** that blocks re-derivation under variant IDs (`CHANGELOG.md:65`).
- It is simulated read-only by `what_if_retract` (`reasons/api.py:512-548`) precisely because it is a state transition, not a content edit.

### `rewrite` — two distinct domains, neither is a truth-value flip

There is **no `rewrite` symbol** in the codebase (definitions: 0). The word appears as (1) a workflow **action string** in premise repair, (2) a **helper** `_rewrite_dependents` used by deduplication, and (3) an unrelated persistence comment.

**Domain A — a reviewer action in premise repair (content corrective).**
`reasons/repair_premises.py:3-42` defines the reviewer protocol: for each flagged premise, choose `rewrite` or `retract`. `rewrite` means the source *supports a corrected version of the claim*, and requires `corrected_text` to be a "complete replacement" (`repair_premises.py:42`). It is applied in `reasons/api.py:4360` ("rewrites premises to match source material or retracts them"), `reasons/api.py:4427` (`if action == "rewrite" and r.get("corrected_text")`), and the count is reported at `reasons/api.py:4445`. The CLI mirrors this at `reasons/cli.py:2747,2756`.

Semantics: **change what the belief says so it remains supportable** — a light-touch correction that keeps the belief rather than pulling it.

**Domain B — `_rewrite_dependents(net, old_id, new_id)` (justification retargeting).**
Defined at `reasons/api.py:4925`, called at `reasons/api.py:5071` and `:5165`. It rewrites dependents' inlist/outlist references from a retiring node id to a surviving canonical id. This is invoked from `apply_dedup_plan` whose docstring is explicit: *"Apply a reviewed dedup plan: rewrite justifications and retract duplicates"* (`reasons/api.py:5143`). The test at `tests/test_derive.py:750-768` states the intent directly: *"Derived beliefs that depended on a retracted duplicate survive via rewrite"* and *"The derived belief should still be IN (rewrite saved it)"*. `tests/test_dependents_integrity.py:345-372` further asserts it cleans up the dependents index (`inlist` and `outlist`).

Semantics: **change what a belief depends on** so that a subsequent retraction of the old id doesn't cascade that belief OUT.

**Domain C — false positive.** `reasons/storage.py:221` ("Clear and rewrite (simple strategy for small networks)") is persistence implementation, not TMS semantics — ignore it.

### The semantic relationship

| Dimension | `retract` | `rewrite` (content) | `rewrite` (dependents) |
|---|---|---|---|
| Locus | truth value of one node's IN/OUT | text of one node | antecedent/outlist references in *other* nodes |
| Cascades | yes, to all dependents | no | no (it *prevents* a cascade) |
| Reversible via | `assert` | not directly (original text lost unless proposed) | no (new edge sticks) |
| First-class API | `retract_node` (`reasons/api.py:473`) | action string only (`repair_premises.py`) | private helper `_rewrite_dependents` (`reasons/api.py:4925`) |
| MCP tool | yes (`reasons/mcp_server.py:146`) | no dedicated tool | no |

The two domains connect via deduplication, where they are **complementary rather than alternatives**:

1. Retracting a duplicate without rewriting would cascade its dependents OUT (that's what retract *does* — `reasons/api.py:221`).
2. `_rewrite_dependents` (called at `reasons/api.py:5165`) redirects those dependents' references to the surviving id **first**, so the subsequent retract has no live dependents to kill (`tests/test_derive.py:768`).

So the domain rule is: **`retract` changes *whether* a belief counts; `rewrite` changes *what a belief says* or *what it rests on*, without flipping its own IN/OUT status.** In premise repair they are mutually exclusive choices; in dedup, rewrite is the *precondition* that makes a retract non-destructive.

### What the observations do not cover

Several important details are not visible and shouldn't be assumed:

- The body of `_rewrite_dependents` (`reasons/api.py:4925`) is not shown — I know it rewrites inlist and outlist and cleans the dependents index from the tests (`tests/test_dependents_integrity.py:345-372`), but not which other reference sites (e.g. `supporting_justification`) are touched.
- The apply path **after** `reasons/api.py:4445` (what a `rewrite` action actually does to the node: mutate text in place? supersede via a proposal? add a new node + retract old?) is not observed. `PROPOSALS_SCHEMA` in `reasons/storage.py` has an `action` CHECK of `('retract','supersede','add','nogood')` with no `rewrite` — from which I'd guess rewriter outputs are routed through `supersede`/`add` proposals, but this is not confirmed here.
- The ordering/atomicity inside `apply_dedup_plan` (`reasons/api.py:5143`) — I see two `_rewrite_dependents` call sites (`:5071`, `:5165`) but not the sequencing with respect to the actual retract of the duplicate.
- `Network.retract` (`reasons/network.py`) — the actual propagation algorithm, cascade-logging, and `retracted_at`/`_retracted` mutation are not in the observations.

## Topics to Explore

- [function] `reasons/api.py:_rewrite_dependents` — Read the body to see exactly which reference sites (inlist, outlist, `supporting_justification`) are rewritten and how the dependents index is kept clean.
- [function] `reasons/api.py:apply_dedup_plan` — Confirm the ordering and atomicity of rewrite-before-retract, and how it decides the surviving id (`keep`).
- [file] `reasons/repair_premises.py` — The full reviewer protocol and how the apply step mutates a premise (in-place update, supersede proposal, or add+retract).
- [function] `reasons/api.py:retract_node` — Full cascade mechanics, `retracted_at`/`_retracted` mutation, and how it contrasts with `what_if_retract`.
- [file] `reasons/network.py` — The core TMS propagation (`Network.retract`, `retract`/`assert` and restoration) that the API wraps.

## Beliefs

- `retract-is-a-truth-value-transition` — `retract_node` (`reasons/api.py:473`) flips a node to OUT and cascades to dependents via `net.retract` (`reasons/api.py:489`); it does not delete the node.
- `retract-preserves-restorability` — Retraction records `retracted_at` (schema in `reasons/__init__.py:49`) and is reversible via `assert` (`.claude/skills/reasons/SKILL.md:79`).
- `rewrite-is-not-a-top-level-api` — No symbol named `rewrite` has a function definition; `rewrite` exists only as a reviewer action string (`reasons/repair_premises.py:32`) and inside `_rewrite_dependents` (`reasons/api.py:4925`).
- `rewrite-dependents-enables-surviving-dependents` — `_rewrite_dependents(net, old_id, new_id)` redirects dependents' references so they remain IN when `old_id` is retracted (`tests/test_derive.py:750-768`).
- `premise-repair-rewrite-requires-corrected-text` — The repair protocol requires `corrected_text` to be a complete replacement when `action == "rewrite"` (`reasons/repair_premises.py:42`).
- `dedup-implements-rewrite-then-retract` — `apply_dedup_plan` docstring states it rewrites justifications and retracts duplicates (`reasons/api.py:5143`) and calls `_rewrite_dependents` at `reasons/api.py:5165`.
