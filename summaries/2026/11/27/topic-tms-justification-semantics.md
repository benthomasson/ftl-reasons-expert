---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: What antecedents/outlist/label mean and why dependents are included in the prompt for cascade reasoning.

# Antecedents, Outlist, Label — and Why `dependents` Shows Up in Cascade-Reasoning Prompts

## The three fields on a `Justification`

Every `Justification` in this codebase carries three fields that describe *why* a node is believed (see `reasons/__init__.py:20-22`):

```python
antecedents: list[str] = field(default_factory=list)  # inlist: must be IN
outlist:     list[str] = field(default_factory=list)  # must be OUT
label:       str                     # human-readable rationale
```

### 1. `antecedents` (aka "inlist")

- **Semantics** — `reasons/__init__.py:11`: *"SL (Support List): node is IN iff ALL antecedents (inlist) are IN."* The justification only holds when **every** antecedent node’s `truth_value` is currently `IN`.
- **ALL vs ANY mode** — The parser at `reasons/api.py:323-328` builds a single `Justification` with the full antecedent list for the default (ALL) mode. If `any_mode` is on, it fans out into one SL justification *per* antecedent (`api.py:325-326`), so the conclusion survives as long as *at least one* premise holds.
- **Namespace handling** — When nodes are added under an agent namespace, an `active_id` premise is prepended to every antecedent list (`api.py:352-355`), then all antecedents are namespace-resolved via `_resolve_namespace` (`api.py:367`).
- **Effect on truth** — `api.py:499-500` (`still_in = [a for a in j.antecedents if a in net.nodes and net.nodes[a].truth_value == "IN"]`) checks the antecedents when re-evaluating a node.

### 2. `outlist` (aka "unless")

- **Semantics** — `reasons/__init__.py:12, 15, 21`: *"AND ALL outlist nodes are OUT."* An outlist node is a **blocker**: the justification holds only while every outlist node is `OUT`.
- **Why it’s needed** — It’s the mechanism for non-monotonic reasoning: *"believe X unless Y is [IN]"* (`__init__.py:15`). Used in code that wants “this conclusion, unless we’re in agent-inactive mode” (`import_agent.py:6, 535`) or “retract old node via outlist” (`api.py:1187`).
- **Evaluation** — `api.py:1590-1592`:
  ```python
  outlist_ok = all(
      o in net.nodes and net.nodes[o].truth_value == "OUT"
      for o in j.outlist
  )
  ```
- **Defeat mechanism** — `api.py:1331-1394` (`defeat_justification`) adds a defeater ID to an existing justification’s `outlist` and registers the reverse edge `defeater.dependents.add(node_id)`. If the defeater is later retracted, the outlist entry is satisfied again and the gated belief flips back IN — the reversible “cascade restoration” described at `api.py:519`.

### 3. `label`

- **Purpose** — A short, human-readable justification label. It is threaded through construction (`api.py:326, 331, 334, 361, 466`), persisted as `justifications.label TEXT DEFAULT ''` (`reasons/storage.py`), and echoed back in exports (`api.py:899`).
- **Where a label value actually comes from** — The derive pipeline asks the LLM to produce one: `derive.py` output format says `- Label: <brief justification rationale>` for each proposed derivation. So `label` is the LLM-authored *why* that ends up attached to the resulting justification.

## Why `dependents` exists — and how it relates to the prompt

`dependents` is a **reverse index** on `Node` (`reasons/__init__.py:38`):

```python
dependents: set[str] = field(default_factory=set)  # reverse index
```

If node `A` appears in node `B`’s antecedents *or* outlist, then `B` is registered under `A.dependents`. This is confirmed by `tests/test_dependents_integrity.py:57-75`:

- `test_single_dependency` — antecedent edge ⇒ `a.dependents == {"b"}`
- `test_outlist_dependency` — **outlist edges also count**: a node with `outlist=["a"]` adds itself to `a.dependents`
- `test_both_antecedent_and_outlist` — both sides of a justification contribute

The population code lives at `api.py:2579-2582` and `api.py:2650-2653` (add-node / add-justification paths) and `api.py:1394, 1528, 1630` (defeat paths). Consistency is enforced by `Network.verify_dependents()` and repaired by `_rebuild_dependents()` (`tests/test_dependents_integrity.py`).

### Why both antecedents *and* outlist contribute to `dependents`

A cascade in this system is not just “lost support.” There are **two directions** the cascade can go:

1. **Antecedent goes OUT** ⇒ the justified node has lost a support and cascades OUT (`api.py:221, 518`).
2. **Outlist node goes IN** ⇒ the *blocker* is now present, so the gated belief cascades OUT (`api.py:586`: *"outlist-gated beliefs that lose their justification when this node goes IN"*).
3. **Outlist node goes OUT** ⇒ the blocker is removed and the gated belief cascades back IN (`api.py:519`: *"restoration from outlist — gated beliefs whose blocker is removed"*).

Because all three cases require “find all nodes whose truth may be affected by this node,” the reverse index must include both antecedent edges **and** outlist edges. Omitting outlist edges would silently break restoration semantics. `_cascade_depth` (`api.py:737-745`) and the public `what_if_retract` / `what_if_assert` helpers (`api.py:475-720`) walk `node.dependents` to compute this.

### Why `dependents` needs to be visible when building prompts for cascade reasoning

The derive/retract prompts are asking an LLM to make decisions whose consequences are defined *by the reverse index*:

- **Prompt context shows what will cascade.** Node detail formatting at `api.py:3298-3299` adds `**Depended on by:** {', '.join(sorted(node.dependents))}`, and `api.py:3023-3028` describes building “neighbors (dependencies and dependents) formatted as readable markdown.” This is the raw material a retraction/derive prompt needs so the model can reason about downstream impact — otherwise it only sees *upstream* antecedents and has zero visibility into what it is about to knock over.
- **Proposal impact is precomputed from `dependents`.** In the proposal pipeline (`api.py:4510-4524`) the code first enriches every node with `nodes[nid]["dependents"] = sorted(net.nodes[nid].dependents)` (line 4514) and then calls `what_if_retract(p["id"])` for each pending proposal, which internally walks `dependents`. The result is embedded as `"cascades"` (`api.py:4519-4531`).
- **The derive prompt explicitly frames cascade semantics.** `derive.py` states: *"When a base premise is retracted, all derived conclusions that depend on it cascade OUT automatically. This is the key value."* To propose *safe, load-bearing* antecedents, the LLM needs both directions — what belief is supported by them (antecedents) and what would break if they go (dependents) — so a retract what-if isn’t a surprise.
- **Restoration is a first-class outcome.** Because outlist edges also feed `dependents`, prompts that show dependents let the model predict *restorations* (gated beliefs flipping back IN) as well as failures. Without dependents in the prompt, the model would only see monotonic-style “who supports me” data and miss non-monotonic cascade behavior entirely.

**Summary of the reason dependents are included:** `dependents` is the only place the transcript of *downstream* consequences lives. Antecedents tell you why you believe X; dependents tell you *what else changes if X flips* — which is exactly the question cascade reasoning is trying to answer.

### One caveat

The observations show the mechanism (reverse index, `_cascade_depth`, `what_if_*`, `**Depended on by:**` markdown) but do **not** show the final prompt template that consumes the enriched node dicts (`api.py:4510-4514`) or the retract/proposal prompt itself. The evidence that `dependents` reaches a prompt is indirect — via the formatter at `api.py:3023-3299` and the enrichment step at `api.py:4514` — so the exact sentence-level wording of that prompt lives outside this slice. `reasons/derive.py` is the best-visible example of a prompt that *does* include antecedent/outlist per conclusion, but it does not directly list `dependents` for each derived node.

## Topics to Explore

- [function] `reasons/api.py:what_if_retract` — Walks `node.dependents` to compute cascade OUT/IN; the canonical consumer of the reverse index during reasoning
- [function] `reasons/api.py:defeat_justification` — Appends to `justification.outlist` and registers the defeater in `dependents`; shows how outlist edges become cascade edges
- [file] `reasons/network.py` — Home of `add_justification`, `retract`, `assert_node`, `verify_dependents`, `_rebuild_dependents`; every mutation path that must keep the reverse index consistent
- [function] `reasons/api.py:format_node_markdown` (around line 3290) — Emits `**Depended on by:**`; this is the piece of code that injects dependents into LLM-facing context
- [general] `derive-prompt-assembly` — The pipeline in `reasons/derive.py` that samples IN beliefs, builds `_build_derived_section`, and turns LLM proposals back into GATE/DERIVE justifications with `label`

## Beliefs

- `antecedents-requires-all-in-for-sl` — An SL justification evaluates true only if every antecedent in `justification.antecedents` has `truth_value == "IN"` (unless `any_mode` fans it out into one-per-antecedent justifications).
- `outlist-requires-all-out` — A justification holds only if every outlist node is `OUT`; a single IN outlist node causes the gated belief to cascade OUT (checked at `api.py:1590-1592`).
- `any-mode-implements-convergent-evidence` — Setting `any_mode` splits a multi-antecedent SL justification into one SL justification per antecedent at `api.py:325-326`, giving ANY-semantics over convergent evidence.
- `dependents-index-includes-outlist-edges` — `Node.dependents` is populated for both antecedent references and outlist references, as enforced by `test_outlist_dependency` and `test_both_antecedent_and_outlist` in `tests/test_dependents_integrity.py`.
- `defeat-justification-causes-restorable-cascade` — `defeat_justification` (`api.py:1331-1394`) appends the defeater to `justification.outlist` and registers it in `defeater.dependents`, so retracting the defeater flips the gated belief back IN (restoration cascade at `api.py:519`).
