---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Useful for understanding the semantics of `antecedents`, `outlist`, and `label` in this codebase.

# Semantics of `antecedents`, `outlist`, and `label`

A quick disambiguation first: `reasons_lib/derive.py` and `reasons_lib/verify.py` are one-line re-export shims (`from reasons.derive import *`). The empty `grep_antecedents_reasons_lib` / `grep_outlist_reasons_lib` / `grep_label_derive` results confirm all the actual implementation lives in the sibling `reasons/` package, not in `reasons_lib/`. Everything below is inferred from `tests/test_outlist.py`, `tests/test_derive.py`, and the documentation strings in `.claude/skills/reasons/SKILL.md`, `CLAUDE.md`, `README.md`, and `CHANGELOG.md`.

## `antecedents` — the positive support list ("inlist")

From `CLAUDE.md:11` and `SKILL.md:33`, an SL (Support List) justification is satisfied **iff every antecedent is IN**. `README.md:9` says the same thing. Tests encode this precisely:

- `tests/test_outlist.py:40-50` (`test_outlist_with_inlist`) — `X` requires `A` IN and `Y` OUT; if `A` is OUT, `X` is OUT even if `Y` is OUT (`test_outlist_with_inlist_both_fail`, lines 52-63). So antecedents are **AND-ed** within a single justification.
- `tests/test_derive.py:88-101` (`test_get_depth`) — justification dicts carry `{"antecedents": [...]}` and `_get_depth` walks them to compute derivation depth. This is the same field used to compute `stats["max_depth"]` in `test_build_prompt_basic` (line ~57).
- `tests/test_derive.py:262-273` (`test_validate_proposals_missing_antecedent`) — proposals whose antecedents reference unknown nodes are dropped; `skipped[0][1]` contains the missing id. Antecedents therefore **must already exist** (unlike outlist — see below).
- `tests/test_derive.py:186-201` (`test_parse_proposals_derive`) — LLM text `- Antecedents: agent-a:knows-auth, agent-b:knows-gateway` parses into `p["antecedents"] == [...]`; backticks are stripped (`test_parse_proposals_strips_backticks`, lines 246-259).

Note the plural "inlist" terminology from `test_outlist.py:2-3` — Doyle's original justification theory. Antecedents are the **inlist**; `outlist` is its negative counterpart.

## `outlist` — the non-monotonic "unless" list

From `README.md:15` and `SKILL.md:56`, outlist implements **"believe X unless Y is believed."** `CLAUDE.md:11` states the full rule: an SL justification is valid **iff every antecedent is IN AND every outlist member is OUT.**

Key semantics, all from `tests/test_outlist.py`:

| Behavior | Test | Lines |
|---|---|---|
| Valid when outlist node is OUT | `test_outlist_valid_when_out` | 18-27 |
| Invalid when outlist node is IN | `test_outlist_invalid_when_in` | 29-37 |
| Absent outlist node counts as OUT (satisfied) | `test_outlist_node_not_in_network` | 61-69 |
| **All** outlist members must be OUT (conjunction) | `test_multiple_outlist_nodes` | 110-127 |
| Asserting an outlist node cascades retraction | `test_assert_outlist_node_retracts_dependent` | 71-88 |
| Retracting restores the dependent | `test_retract_outlist_node_restores_dependent` | 90-106 |
| Cascade propagates downstream (X→Z) | `test_outlist_cascade_through_chain` | 108-127 |
| Explain output exposes `outlist` / `violated_outlist` | `test_explain_shows_outlist_on_valid` / `_violated_outlist` | 158-180 |
| Alternative justification can keep the node IN | `test_alternate_justification_survives_outlist` | 133-153 |

Two important contrasts with antecedents:

1. **Missing node is OUT, not an error.** `test_outlist_node_not_in_network` (line 61) shows an outlist reference to a nonexistent id is treated as satisfied. Antecedents, by contrast, must exist (`test_validate_proposals_missing_antecedent`).
2. **Cascade semantics.** `net.assert_node("y")` returns a `changed` set that includes poisoned dependents (`test_outlist.py:85-87`), and `net.retract("y")` restores them (lines 102-104).

`CHANGELOG.md:33` gives the design rationale: during `import-agent`, "active premise [was] moved from antecedent to outlist so per-belief retraction works independently" — i.e., controlling whether a belief is *trusted* logically belongs in the outlist, not in the mandatory antecedent list. `CHANGELOG.md:131` and `CLAUDE.md:57` (`--unless y`) mirror the same idea. `SKILL.md:91` notes that `reasons challenge TARGET` creates an IN node and adds it to the target's outlist — the primary CLI operation for defeat.

At the API layer, `--unless` maps directly to `outlist` (`tests/test_outlist.py:210-251`, `TestOutlistAPI`): `api.add_node("x", ..., unless="y", db_path=db)` produces an SL node with `outlist=["y"]`, and `test_unless_only_no_sl` (lines 240-251) asserts `result["type"] == "SL"` — so `--unless` alone synthesizes an SL justification with empty antecedents.

## `label` — free-form provenance/annotation on a justification

`label` is a **per-justification** annotation (not a node attribute) that describes *why* this justification holds. It is separate from a node's `text` description.

- CLI usage: `SKILL.md:61` — `reasons add node-id "Description" --sl dep-a --source ... --label "why this justification holds"`; `README.md:46` — `--label "observed from source"`.
- Persistence: `tests/test_outlist.py:185-205` (`TestOutlistPersistence::test_round_trip`) creates a justification with `label="default unless Y"`, saves through `Storage`, reloads, and asserts `j.label == "default unless Y"`. So label survives the SQLite round-trip as a first-class justification field.
- Provenance use-case: `tests/test_derive.py:32-45` (`agent_network` fixture) — imported nodes carry `label="imported from agent: agent-a"`, i.e., label records *where a belief came from*, not what it means.
- Explain rendering: `README.md:82-83` shows explain output with justification text followed by `[observed from source]` in brackets — this bracket appears to be the label rendered inline next to a satisfied SL justification.
- LLM proposal parsing: `tests/test_derive.py:199-201` (`test_parse_proposals_derive`) parses `- Label: cross-agent authentication flow` into `p["label"]`. So when `derive` proposes new nodes, the label is the human-readable provenance tag that lands on the resulting justification. `test_parse_proposals_multiple` (lines 222-236) shows labels distinguish otherwise-identical proposals.

## How they fit together

Within one `Justification` object (as constructed in tests and parsed from LLM proposals):

```
valid(j)  ⇔  ∀ a ∈ j.antecedents: truth(a) == "IN"
           ∧  ∀ o ∈ j.outlist:     truth(o) == "OUT"
```

A node is IN iff **some** justification is valid (`test_alternate_justification_survives_outlist`, `test_outlist.py:133-153`). So the semantics are **AND within a justification, OR across justifications**. `label` does not participate in truth evaluation — it travels with the justification purely as metadata for human/AI readers (and for the `reasons show` / `explain` UI).

`mode` in `tests/test_derive.py:204-221` (`test_parse_proposals_with_mode_any`) is a proposal-time hint: the CLI defaults to `mode="all"` (all antecedents required), but proposals can request `mode="any"` (disjunctive support), which is emitted to the parser but not yet shown to alter the underlying SL semantics in these observations.

## What's missing from the observations

- The actual `Justification` class (presumably `reasons/__init__.py`) is never shown — field defaults, validation, and construction are inferred only from test usage.
- `Network.assert_node` / `Network.retract` / `Network.explain` implementations are not read; the cascade and explain output shape are inferred from test expectations (lines 84-87, 102-104, 158-180).
- `reasons/derive.py` (real module) is not read; only `reasons_lib/derive.py` (the shim) and its tests. `build_prompt`, `parse_proposals`, `validate_proposals`, `apply_proposals`, and `find_similar_out` signatures/semantics are visible only through test call sites in `tests/test_derive.py:1-300`.
- The `Storage` schema is not shown, so whether `label` is a column on the justifications table or stored inside a serialized blob is unknown.

## Topics to Explore

- [file] `reasons/__init__.py` — Defines the `Justification` class with `type`, `antecedents`, `outlist`, `label`; the missing source-of-truth for these three fields.
- [function] `reasons/network.py:Network.assert_node` — Implements the outlist cascade that returns the `changed` set verified in `tests/test_outlist.py:84-87`.
- [function] `reasons/derive.py:parse_proposals` — Real parser for LLM proposals; maps `Antecedents:` / `Unless:` / `Label:` / `Mode:` lines onto justification fields (tests at `tests/test_derive.py:186-259`).
- [file] `reasons/storage.py` — SQLite schema; confirms whether `label` and `outlist` are separate columns or serialized, given `test_round_trip` at `tests/test_outlist.py:185-205`.
- [general] `inlist-outlist-precedence` — Whether `mode="any"` interacts with outlist semantics (currently not exercised by any observed test).

## Beliefs

- `justification-requires-all-outlist-nodes-out` — An SL justification is valid only when every node in its `outlist` is OUT, per `CLAUDE.md:11` and `tests/test_outlist.py:71-88`.
- `absent-outlist-node-is-treated-as-out` — An outlist reference to a node not in the network counts as satisfied; see `test_outlist_node_not_in_network` at `tests/test_outlist.py:61-69`.
- `antecedents-must-exist-outlist-need-not` — `validate_proposals` rejects proposals with missing antecedents but outlist references are unvalidated; contrast `tests/test_derive.py:262-273` with `tests/test_outlist.py:61-69`.
- `multiple-justifications-are-or-ed` — A node is IN if any one of its justifications is valid; an alternate justification survives a violated outlist (`test_alternate_justification_survives_outlist`, `tests/test_outlist.py:133-153`).
- `label-persists-through-sqlite-round-trip` — `Storage.save`/`load` preserves the per-justification `label` verbatim (`test_round_trip`, `tests/test_outlist.py:185-205`).
- `reasons-lib-package-is-a-shim-over-reasons` — Every file under `reasons_lib/` observed re-exports from `reasons.*`; zero grep hits for `antecedents`/`outlist`/`label` inside `reasons_lib/`.
