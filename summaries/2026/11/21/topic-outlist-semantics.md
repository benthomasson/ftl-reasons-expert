---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The `retract_reason` / `_retracted` legacy representation and the `outlist` mechanism implied by the test docstrings; understanding both clarifies why the migration exists.

# Legacy Retraction vs. Graph-Native Defeat

## The two representations

The codebase carries **two different ways to express "this belief is OUT"**, and the migration exists to convert nodes from the first to the second.

### 1. Legacy: `retract_reason` / `_retracted` metadata side-channel

When a node is retracted through the old path, its **truth value flips to OUT** and the reason is stashed in node metadata:

- `reasons/api.py:1269` and `reasons/api.py:1310` write `meta["retract_reason"] = reason`.
- The companion flag `_retracted` (mentioned in the migration docstring at `reasons/api.py:1557`: "…would be satisfied if not for the `_retracted` flag") is what actually suppresses the node; `retract_reason` is the human-readable explanation.
- The CLI mirrors this via `api.set_metadata(r["id"], "retract_reason", reason, ...)` at `reasons/cli.py:2475`, and the reason is surfaced for display in `reasons/cli.py:553-554`, `reasons/compact.py:100-102`, and `reasons/export_markdown.py:54,103`.

The crucial property: **the node's justifications are untouched**. They still list their antecedents and they still evaluate as satisfied. The node is OUT *only* because a metadata flag says so. That is why the migration docstring (`reasons/api.py:1554-1557`) specifically targets nodes "**whose justifications would be satisfied if not for the `_retracted` flag**."

This legacy encoding is a one-way, static annotation. It has no first-class object, no edge in the belief graph, and no participation in belief revision. You cannot "retract the retraction."

### 2. Graph-native: `outlist` / defeater beliefs

The non-monotonic machinery uses **outlist** on `Justification` instead:

- `reasons/__init__.py:21` — `outlist: list[str] = field(default_factory=list)  # must be OUT`.
- `reasons/__init__.py:12-15` — a justification is valid iff all `antecedents` are IN **AND all `outlist` nodes are OUT**. "The outlist enables non-monotonic reasoning: 'believe X unless Y is believed.'"

The semantics, spelled out in `tests/test_outlist.py`:
- `test_outlist_valid_when_out` — Y OUT → X IN.
- `test_outlist_invalid_when_in` — Y IN → X OUT.
- `test_outlist_node_not_in_network` — absent outlist node is treated as OUT.
- `test_assert_outlist_node_retracts_dependent` / `test_retract_outlist_node_restores_dependent` — asserting Y flips X OUT; retracting Y **restores X IN**, automatically, via the cascade.
- `test_outlist_cascade_through_chain` — changes propagate through dependents.
- `test_alternate_justification_survives_outlist` — other justifications keep the node alive.
- `test_explain_shows_outlist_on_valid` / `..._violated_outlist` — the outlist is visible in `explain()`.

You can produce an outlist directly (`reasons/api.py:302,320-334`, via the `unless=` argument) or by *defeating* an existing justification: `defeat_justification` (`reasons/api.py:1331-1334`) "Defeat a justification by adding a defeater belief to its outlist," which appends to `justification.outlist` at `reasons/api.py:1392-1393` and `reasons/api.py:1526-1527`.

**Key difference:** the defeater is a *node in the graph*. Because it's a node, it can be asserted, retracted, classified, explained, and cascaded through — exactly like any other belief. That is what makes the retraction *reversible* and *traceable*.

## Why the migration exists

`migrate_retract_to_defeaters` (`reasons/api.py:1549`) exists to eliminate the split-brain: after migration, there is exactly **one** representation of OUT-ness — the outlist — so:

1. **Reversibility.** A migrated retraction becomes a defeater node (`migrated-retraction-{nid}-j{idx}`, `reasons/api.py:1612`, `1614`). Retracting it restores the original node to IN — demonstrated in `test_reversibility` (`tests/test_migrate_defeaters.py`). The legacy metadata approach cannot do this: there's no object to retract.
2. **Composability with belief revision.** Metadata flags don't participate in the cascade, `explain`, or the "restoration from outlist" logic noted at `reasons/api.py:519` / `586`. Outlist edges do.
3. **Auditability.** The defeater carries structured metadata — `defeater_type: "migrated-retraction"` and `defeats_node: "derived"` (`reasons/api.py:1621`; asserted in `test_basic_migration`). This can be filtered/classified (see `tests/test_classify_defeaters.py:69,131`), something a free-text `retract_reason` cannot.
4. **Semantic honesty.** With legacy metadata the node's justifications still look satisfied, so any graph traversal that looks at justifications sees an inconsistency. After migration the justifications are genuinely blocked by an outlist entry, matching the node's actual OUT status.

## How the migration works (walkthrough)

From `reasons/api.py:1549-1649` plus `tests/test_migrate_defeaters.py`:

1. **Candidates:** iterate OUT nodes, read `retract_reason` from metadata (`reasons/api.py:1576-1577`). Nodes without it are skipped (`test_skip_no_retract_reason`).
2. **Justification check:** for each justification, verify the antecedents are satisfied **and** `outlist_ok` holds — `all(...)` over `j.outlist` at `reasons/api.py:1590-1592`. If the antecedent is already OUT, the node would be OUT anyway; migration skips it (`test_skip_unsatisfied_justification`). This is the "would be satisfied if not for `_retracted`" test from the docstring.
3. **Defeater creation:** build a new node `migrated-retraction-{nid}-j{satisfied_idx}` (`reasons/api.py:1612`) with metadata `defeater_type="migrated-retraction"` and `defeats_node=nid` (`reasons/api.py:1621`), and append its id to the justification's `outlist` so the node stays OUT through the graph-native path.
4. **Cleanup:** pop `retract_reason` and `_retracted` from the node's metadata (`reasons/api.py:1636-1637`), confirmed by `test_metadata_cleared`.
5. **Dry run:** `dry_run=True` returns candidates with no `defeater_id` and leaves metadata intact — `test_dry_run`.
6. **Filtering / errors:** `node_ids` restricts scope (`test_node_ids_filter`); missing ids land in `errors` (`test_error_on_missing_node`).
7. **Apply:** commit only when `not dry_run and migrated` (`reasons/api.py:1646`). CLI entry point is `cmd_migrate_defeaters` (`reasons/cli.py:285-318`), registered as `migrate-defeaters` (`reasons/cli.py:3155-3156`, dispatched at `reasons/cli.py:3934`); it refuses `--pg` because there's no `PgApi` implementation (`reasons/cli.py:287`).

## What's missing from the observations

- The actual write of `_retracted = True` is never shown; only `reasons/api.py:1269/1310` (write of `retract_reason`) and `1637` (removal of `_retracted`). Presumably it's set nearby, but the code isn't in the snapshot.
- `find_storage_retraction_usage` and `find_derive_backtracking_usage` returned zero matches — so this metadata is **not** consumed by `storage.py` or the derive/backtracking path; it's purely a `reasons/api.py` + CLI/export concern. Worth verifying against the real files.
- The consumer at `reasons/api.py:3699-3704` (re-reading `retract_reason` and republishing it) isn't explained by the observations.

## Topics to Explore

- [function] `reasons/api.py:migrate_retract_to_defeaters` — Read the full body (1549–1649) to see the exact `outlist_ok`/antecedent check and how the defeater is attached.
- [file] `reasons/__init__.py` — The `Justification` dataclass definition; confirms `outlist` is a first-class field with "must be OUT" semantics.
- [file] `tests/test_outlist.py` — The canonical suite pinning down outlist semantics (reversibility, cascading, missing nodes, multiple justifications).
- [function] `reasons/api.py:defeat_justification` — How a defeater node gets appended to a justification's `outlist` outside of migration (lines ~1331–1393).
- [general] `retracted-flag-writers` — Locate where `_retracted` is actually set (not in the observations); those are the remaining legacy write sites.

## Beliefs

- `outlist-mechanism-enables-non-monotonic-retraction` — A `Justification` is valid only when all antecedents are IN and all `outlist` nodes are OUT, so asserting an outlist node retracts the dependent and retracting it restores the dependent.
- `retract-reason-is-metadata-side-channel` — The legacy `retract_reason`/`_retracted` representation marks a node OUT via metadata while leaving its justifications satisfied.
- `migrate-retract-to-defeaters-replaces-metadata-with-outlist-edges` — `migrate_retract_to_defeaters` clears `retract_reason`/`_retracted` and instead appends a `migrated-retraction-*` defeater node to each satisfied justification's `outlist`.
- `migration-skips-nodes-with-unsatisfied-antecedents` — Nodes whose antecedent is already OUT are not candidates because they would be OUT regardless of the retraction.
- `migrated-defeater-is-reversible-not-static` — Retracting the generated `migrated-retraction-*` node restores the original claim to IN, which the legacy metadata representation cannot express.
- `migration-requires-satisfied-and-outlist-ok-justification` — Only justifications that pass both the antecedent check and `outlist_ok` at `reasons/api.py:1590` produce a defeater.
