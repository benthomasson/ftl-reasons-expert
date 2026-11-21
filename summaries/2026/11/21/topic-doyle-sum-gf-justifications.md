---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Background theory: why SUM/GF justifications model abstraction and what semantics distinguish them from generic SL

## Background theory: SUM/GF abstraction justifications vs. generic SL

### What the code actually establishes

The repository only **mentions** SUM/GF in one place: the module docstring of `tests/test_summarize.py` (line 3):

> "Doyle's SUM/GF justifications let you abstract away internal details."

The docs scan finds **zero** SUM/GF matches, and the implementation surface found here only exercises `SL` and `CP` justifications (`tests/test_add_justification.py`, `reasons/derive.py`). The `Justification` class is declared at `reasons/__init__.py:7`, but its body is not in the observations. So the theory below is a faithful description of Doyle-style justification semantics and the abstraction role the test docstring attributes to SUM/GF, but the **exact field-level semantics used by this codebase are not visible** — treat the implementation mapping as inferred, not confirmed.

### Generic SL semantics as implemented here

From `tests/test_add_justification.py` and `reasons/derive.py`:

- A justification has `type`, `antecedents`, and `outlist`; `reasons/storage.py` persists these as `type`, `antecedents_json`, and `outlist_json` in the `justifications` table.
- A node is `IN` if **some** justification is valid. Validity for an SL/defeasible edge means: all antecedents are `IN` **and** every outlist node is `OUT`.
- Multiple justifications are disjunctive: `test_added_justification_keeps_node_in_after_retract` shows a node staying `IN` after an antecedent is retracted because a second SL justification still holds.
- `test_add_justification_with_outlist` proves the outlist direction: `b` remains `OUT` while `enemy` is `IN`, and flips `IN` only after `enemy` is retracted.
- Adding a justification registers the target as a **dependent** of both antecedents and outlist nodes (`test_add_justification_registers_dependents`), which is what drives propagation cascades.
- `reasons/derive.py`'s prompt describes this same semantics operationally: SL rules are positive support; outlist-gated beliefs hold "unless" a negative node is `IN`.

This is generic, extensional, truth-maintenance dependency: every edge points at a concrete node ID, and propagation walks those IDs. `CP` appears as a type in tests, but the observations do not show it behaving differently from `SL` at propagation level.

### Why plain SL does not model abstraction

An SL edge is **flat and transparent**. If a consumer cites an abstraction node, and that node cites the internal nodes, then transitive retraction reaches the consumer whenever any internal node changes. Consequences:

1. **Fine-grained dependency leakage.** There is no way to say "this node stands for this whole sub-derivation"; changes to private internals propagate to public consumers.
2. **No encapsulation boundary.** The abstraction has no interface distinct from its implementation — consumers are coupled to every antecedent transitively.
3. **Defeaters are local to justifications, not to abstractions.** An outlist attached to one internal SL edge is a private implementation detail, yet it can flip the public node.
4. **No distinction between "supported" and "grounded."** SL says a node is supported by named antecedents; it does not separately record the obligation that an abstracted derivation remains grounded in premises/assumptions.

The `_get_depth` cycle guard in `reasons/derive.py` is a small signal that well-foundedness/cycles are a real concern in this propagation model — a relevant pressure point for any abstraction mechanism.

### What SUM/GF add conceptually

Doyle's TMS justification vocabulary is the theoretical background: SL justifications encode support lists (positive antecedents plus defeaters), while other types encode richer proof obligations. The test docstring's claim is that **SUM** and **GF** are the abstraction-oriented pair:

- **SUM (summarization):** a justification that a node is supported by a *summarized* derivation / boundary set rather than by every internal node. This is the abstraction edge: the summary node is justified as a stand-in for a sub-network. Consumers depend on the boundary, not the internals.
- **GF (grounding):** the obligation that the summarized derivation is still grounded — derivable from premises/assumptions through the summarized structure. It is the soundness condition on the abstraction.

Combined, the intended semantics are: **the abstraction is valid iff its summarized derivation is still grounded and its boundary conditions hold.** Internal rewiring that preserves grounding should not invalidate the summary; breaking grounding must invalidate it.

That is exactly what the `summarize` feature in this repo is positioned to produce: `reasons/api.py:1149`, `reasons/network.py:775`, and `reasons/pg.py:1727` all define a "summary node that abstracts over a group of nodes." The `tests/test_summarize.py` docstring frames the theoretical justification for that operation in SUM/GF terms.

### Semantics that distinguish SUM/GF from generic SL

| Dimension | Generic SL | SUM/GF abstraction |
|---|---|---|
| Dependency granularity | Direct node IDs per justification | Summarized/boundary set; internals hidden |
| Validity condition | All antecedents IN and outlist OUT | Grounding of the summarized derivation plus boundary conditions |
| Retraction locality | Any antecedent change cascades | Only grounding/boundary changes cascade to consumers |
| Encapsulation | None | Summary node acts as an interface over implementation nodes |
| Defeater placement | Outlist per justification edge | Defeaters should attach at the abstraction boundary |
| Well-foundedness | Implicit in propagation | Grounding is an explicit obligation |
| Extensibility | Fixed by edge shape | Type-tagged justification variant |

One architectural note from `reasons/storage.py`: the `justifications.type` column is declared `TEXT NOT NULL` with **no CHECK constraint or enum**, so the persistence layer is already type-agnostic. A `SUM`/`GF` justification could be stored without a schema migration — which means type validation, if it exists, must live in `Justification` or propagation code rather than in SQLite.

### What is missing from these observations

To confirm rather than infer the SUM/GF semantics, we would need:

- the body of `Justification` in `reasons/__init__.py` (starting at line 7) — field names, validation, allowed `type` values;
- the propagation branch in `reasons/network.py` / `reasons/pg.py` that evaluates justifications by `type`;
- the test bodies in `tests/test_summarize.py` beyond the line-3 docstring;
- any docs page describing summarize — the docs scan currently returns zero SUM/GF matches.

## Topics to Explore

- [file] `tests/test_summarize.py` — The only SUM/GF reference in the repo; its test bodies reveal the expected abstraction and retraction behavior.
- [file] `reasons/__init__.py` — Contains the `Justification` class defined at line 7; its body determines whether SUM/GF are first-class types or just strings.
- [function] `reasons/network.py:add_justification` — Registers antecedent and outlist dependents and evaluates justification validity; the place where type-specific semantics would branch.
- [file] `reasons/api.py` — Line 1149 creates the summary node that the SUM/GF docstring is explaining; inspect the public surface and how it builds the summary.
- [general] `truth-maintenance-propagation` — How IN/OUT cascades handle multiple disjunctive justifications, outlists, and cycles; the machinery any abstraction justification must plug into.

## Beliefs

- `justification-types-are-not-db-constrained` — `reasons/storage.py` declares `justifications.type` as `TEXT NOT NULL` with no CHECK or enum, so SUM/GF would need code-level validation, not schema enforcement.
- `sl-justification-requires-in-antecedents-and-out-outlist` — `test_add_justification_with_outlist` shows a node stays `OUT` while its outlist node is `IN` and flips `IN` only after that node is retracted.
- `add-justification-registers-antecedent-and-outlist-dependents` — `test_add_justification_registers_dependents` asserts the target node is registered in `dependents` of both antecedent nodes and outlist nodes.
- `multiple-justifications-are-disjunctive` — `test_added_justification_keeps_node_in_after_retract` proves a second SL justification keeps a node `IN` after one antecedent is retracted.
- `sum-gf-abstraction-rationale-only-appears-in-test-docstring` — The only repo match for `SUM|GF` is `tests/test_summarize.py:3`; the docs search returns zero matches, so the abstraction claim is documented only in that test docstring.
