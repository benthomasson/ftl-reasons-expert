---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Background on Doyle/de Kleer TMS semantics — the `IN`/`OUT` truth values and `outlist` vocabulary used throughout these tests are direct borrowings.

# Background on Doyle / de Kleer TMS semantics

## Where the vocabulary comes from

The codebase is explicit about its lineage. `README.md:3` opens: *"An implementation of Doyle's (1979) Truth Maintenance System"*, and `README.md:198` cites the source paper verbatim: **Doyle, J. (1979). A Truth Maintenance System. *Artificial Intelligence*, 12(3), 231–272.** The same attribution recurs at `CLAUDE.md:5`, `reasons/__init__.py:1` ("Reason Maintenance System — data model based on Doyle (1979)"), and `reasons/network.py:3` ("Implements Doyle's (1979) TMS algorithm").

The test suite names Doyle directly when the semantics matter:

- `tests/test_outlist.py:3–4` — *"Doyle's key innovation: SL justifications have both an inlist (must be IN) and an outlist (must be OUT). 'Believe X unless Y is believed.'"*
- `tests/test_network.py:54` — *"Retraction cascades — Doyle's core mechanism."*
- `tests/test_dialectical.py:3` — *"Doyle Section 6: arguments become explicit, challengeable beliefs."*
- `tests/test_summarize.py:3` — *"Doyle's SUM/GF justifications let you abstract away internal details."*

So the `IN`/`OUT` values and the `outlist`/`inlist` terms are not internal inventions — they are the literal vocabulary of Doyle's 1979 paper, reused here so that readers who know the TMS literature can map the code straight onto the paper.

**Caveat about de Kleer:** the task title mentions de Kleer, but the term `de Kleer` / `Kleer` does not appear anywhere in the grep results (only `Doyle`). The code's self-declared ancestor is Doyle (1979). De Kleer's ATMS (1986) is likely the conceptual backdrop for features like `nogood` (`SKILL.md:38`), but the observations do **not** contain a direct de Kleer citation — see "What's missing" below.

## The `IN` / `OUT` truth values

A node is either **IN** (currently believed) or **OUT** (currently not believed). `.claude/skills/reasons/SKILL.md:31–37` acts as a glossary:

- **Premise**: a node with no justifications → **IN** by default (`SKILL.md:31`, example at `:50`).
- **Justified node**: **IN** only while its justification is valid (`SKILL.md:32`).
- **Retraction**: flipping a node OUT cascades to dependents (`SKILL.md:35`).
- **Restoration**: when the retracted node comes back IN, dependents recompute automatically — the key Doyle insight that retracted nodes stay in the DB (`CLAUDE.md:95`, `SKILL.md:36`).

The implementation-level data model lives in `reasons/__init__.py`, whose module docstring (`:1`) and `outlist` field (`:21`, `# must be OUT`) encode the same semantics.

## Justifications: `inlist` vs `outlist`

Doyle's SL justification is characterised in `reasons/__init__.py:12`:

> *"(node is IN if) AND ALL outlist nodes are OUT"*

and at `:15`:

> *"The outlist enables non-monotonic reasoning: 'believe X unless Y is believed'."*

So a justification has two sides:

- **`antecedents` / inlist** — every node must be IN.
- **`outlist`** — every node must be OUT.

This is exactly what `tests/test_outlist.py` exercises end-to-end:

- `test_outlist_valid_when_out` (`test_outlist.py:~25`) — Y retracted, so X is IN.
- `test_outlist_invalid_when_in` (`:~36`) — Y asserted, so X is OUT.
- `test_outlist_with_inlist` (`:~50`) — both an inlist antecedent (`a`) and an outlist blocker (`y`).
- `test_outlist_node_not_in_network` (`:~85`) — a *missing* outlist node is treated as OUT (absent ⇒ not believed), which matches Doyle's reading.
- `test_assert_outlist_node_retracts_dependent` / `test_retract_outlist_node_restores_dependent` (`:~100–130`) — the non-monotonic flip is reversible, mirroring Doyle's claim that belief revision needs no rederivation.

## How `outlist` surfaces at the API layer

The CLI/API parameter is called `unless`, and it maps 1:1 onto `Justification.outlist`:

- `reasons/api.py:302` — *"unless: Comma-separated outlist IDs (must be OUT for justification to hold)"*.
- `reasons/api.py:320` — `outlist = [o.strip() for o in unless.split(",") if o.strip()] if unless else []`.
- `reasons/api.py:326–334` — the outlist is attached when constructing SL/CP justifications ("unless-only" becomes an SL justification with an empty antecedent list, per `:334`).
- `reasons/api.py:368` — namespace resolution for outlist entries.
- `reasons/api.py:437,455,460,466` — the same mapping for the second `add`-style API surface.

So when a test calls `api.add_node("x", "Default X", unless="y", ...)` (`test_outlist.py:~230`), it is creating an SL justification whose `outlist=["y"]` — the same shape as the low-level constructor `Justification(type="SL", antecedents=[], outlist=["y"])` used in the unit tests.

## Reversibility and the "defeat" idiom

Two pieces of Doyle's machinery that show up repeatedly:

1. **Reversible retraction via the outlist itself.** `reasons/api.py:519` and `:586` describe restoration/gating in terms of `outlist` state changes; `:1187` says *"The old node goes OUT via the outlist mechanism (reversible)."* This matches Doyle's point that retraction without discarding the node preserves the ability to restore it (`CLAUDE.md:95`).

2. **Defeat = append to `outlist`.** `reasons/api.py:1331` — *"Defeat a justification by adding a defeater belief to its outlist"* — followed at `:1392–1393` by literally `justification.outlist.append(defeater_id)`. The docstring at `:1334–1335` frames this as *"graph-native TMS semantics rather than metadata strings"*, i.e. the belief network *is* the truth maintenance system, not an annotation layer over one. This is what `tests/test_dialectical.py` (Doyle Section 6) exercises: challenges and defenses become ordinary nodes whose IN/OUT state gates the target through its outlist (`SKILL.md:91–96`).

## What the observations don't tell us

- **No de Kleer citation.** The grep for `Kleer` returns nothing. If the reader wants the ATMS angle specifically, `reasons/` does not document it as such — only `nogood` (`SKILL.md:38`) is closer to ATMS vocabulary.
- **No definitions for `SUM` / `GF`.** They are named in `tests/test_summarize.py:3` but not defined in the shown excerpts; `reasons/summarize.py` and `reasons/network.py` would be needed to see implementation.
- **Only the module docstring of `reasons/network.py` was retrieved** (line 3). The actual propagation algorithm — how it recomputes truth values after a cascade — is not in the observations.
- **The `defeat_justification` function signature** at `api.py:~1331` is inferred from its docstring; the exact name is not shown in the greps.

---

## Topics to Explore

- [file] `reasons/network.py` — Holds the Doyle 1979 algorithm implementation (docstring line 3); the actual truth-value propagation, cascade and restoration logic live here, not in the API layer.
- [file] `reasons/__init__.py` — The `Justification` dataclass, docstrings at lines 12–21, and the `outlist` field (`# must be OUT`) are the canonical in-code definition of Doyle semantics.
- [file] `reasons/api.py` — `unless` → `outlist` translation (lines 302, 320, 326–334) and the defeat-by-outlist-append mechanism (lines 1331–1393); the two main places Doyle vocabulary leaks into the public API.
- [file] `tests/test_dialectical.py` — Explicitly anchored to "Doyle Section 6"; shows how challenge/defend map onto outlist mutations, the connection most readers miss.
- [general] `nogood-vs-doyle` — `SKILL.md:38` introduces nogoods (a set of nodes that cannot all be IN); check whether the resolution machinery follows Doyle's contradiction handling or de Kleer-style nogood records, since the codebase cites only Doyle.

## Beliefs

- `outlist-is-non-monotonic-justification` — An `outlist` requires every listed node to be OUT for the justification to hold, encoding "believe X unless Y" (`reasons/__init__.py:15,21`; `tests/test_outlist.py:3–4`).
- `unless-argument-populates-outlist` — The API `unless=` parameter is split on commas and assigned verbatim to `Justification.outlist` (`reasons/api.py:302,320,326–334,437–466`).
- `retracted-nodes-remain-stored-for-restoration` — Retracted nodes stay in the DB specifically to enable restoration without rederivation, a design point attributed to Doyle (`CLAUDE.md:95`, `SKILL.md:36`).
- `defeat-justification-appends-to-outlist` — Defeating a justification adds a defeater node ID to `justification.outlist` rather than mutating metadata strings, keeping the belief network as the source of truth (`reasons/api.py:1331–1393`).
- `tms-vocabulary-is-doyle-1979-not-de-kleer` — Every "TMS" citation in the codebase points to Doyle 1979 (`README.md:3,198`; `reasons/__init__.py:1`; `reasons/network.py:3`); "de Kleer" has zero grep matches.
