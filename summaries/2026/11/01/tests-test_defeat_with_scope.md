---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_defeat_with_scope.py

# `tests/test_defeat_with_scope.py`

## Purpose

This file validates the **scope-belief defeater mechanism** in the `ftl-reasons` belief-network system. It tests `api.defeat_with_scope()`, a function that performs a *justified* defeat of a belief (flipping its truth value from `IN` → `OUT`) by introducing intermediate "scope beliefs" that articulate *what each premise actually establishes* versus *what it fails to establish*. The defeater itself is a first-class derived node, so the defeat is auditable, retractable, and exportable — not an opaque state mutation.

In the broader project, this is the "why did this belief die?" layer. Other test files (`test_defeat_justification.py`, `test_classify_defeaters.py`) cover simpler or classification-based defeaters; this file covers the most structurally rich case.

## Key Components

| Component | Contract |
|-----------|----------|
| `api.init_db(path)` | Creates a fresh SQLite-backed belief graph. Every test gets isolation via `tmp_path`. |
| `api.add_node(id, text, sl=..., db_path=...)` | Inserts a node. `sl` is a comma-separated list of antecedent node IDs, creating an SL (Set of Links) justification. |
| `api.show_node(id, db_path=...)` | Returns a dict with `truth_value` (`"IN"`/`"OUT"`), `justifications` (list), `metadata` (dict), and `text`. |
| `api.defeat_with_scope(node_id, just_idx, scope_findings, missing_property, defeater_id=..., defeat_reason_type=..., db_path=...)` | Core SUT. Creates scope-belief premises, derives a defeater node from them, and flips the target to OUT. Returns a dict with `node_id`, `justification_index`, `scope_belief_ids`, `defeater_id`, `changed`, `defeat_reason_type`. |
| `api.retract_node(id, db_path=...)` | Flips a node back to OUT (or removes it), triggering cascade re-evaluation of dependents. |
| `api.defeat_justification(node_id, just_idx, reason, defeat_reason_type=..., db_path=...)` | Simpler defeater that doesn't create scope beliefs — tested in the last test. |
| `api.export_network(db_path=...)` / `api.import_json(path, db_path=...)` | Serialization round-trip used to verify structural integrity. |
| `api.add_justification(node_id, sl=..., db_path=...)` | Adds an additional SL justification to an existing node (used in the uniqueness test). |

## Patterns

1. **Arrange-Act-Assert per test.** Every test follows: init DB → add nodes (arrange) → call the API (act) → inspect returned dict + re-fetch nodes (assert). No shared mutable state between tests; `tmp_path` gives each test its own SQLite file.

2. **Black-box API testing.** Tests only call the public `api.*` functions. They never reach into `reasons/storage.py` or `reasons/pg.py` directly. The internal node graph is inspected solely through `show_node`.

3. **Round-trip serialization test.** `test_defeat_with_scope_in_export` ensures the scope-belief + defeater structure survives `export → JSON → import`, guarding against lossy serialization of metadata or justification links.

4. **Parameterized error testing.** `test_defeat_with_scope_errors` bundles four distinct `pytest.raises` blocks in one test, each checking a different failure mode (missing node, no justifications, out-of-range index, empty findings).

5. **Cascade verification.** `test_cascade_through_scope_defeat` and `test_challenge_scope_restores_target` assert that truth-value changes propagate transitively through the dependency graph (D1 → D2, and scope → defeater → target).

## Dependencies

**Imports (this file depends on):**
- `pytest` — test runner, `tmp_path` fixture, `pytest.raises`
- `reasons.api` — the entire public API surface under test
- `json` (local import in `test_defeat_with_scope_in_export`) — serializing the export for the import round-trip

**Imported by (what depends on this file):**
- Nothing in production code imports a test file. It is consumed by the test runner (pytest) and by CI (`/.github/workflows/`).

**Indirect dependencies (through `reasons.api`):**
- `reasons.storage` — node/edge persistence
- `reasons.pg` — the proof-graph (SL justifications, truth-value propagation)
- `reasons.network` — cascade / retraction logic
- `reasons.export_markdown` / internal export helpers

## Flow

Taking `test_defeat_with_scope_basic` as the representative:

```
1. init_db → empty graph
2. add_node("premise-a"), add_node("premise-b")
3. add_node("belief", sl="premise-a,premise-b")
   → belief truth_value = IN (justified by both premises)
4. defeat_with_scope("belief", 0, scope_findings=[...], missing_property="traceability")
   Internally:
     a. Validates node exists, justification index is valid, scope_findings non-empty
     b. For each scope_finding, creates a new IN premise node:
        - text: "establishes X ... does not establish Y"
        - metadata.scope_of = antecedent
        - metadata.for_defeater = defeater_id
     c. Creates a defeater node (IN, derived) with SL justification
        whose antecedents = [scope_belief_1, scope_belief_2]
     d. Links defeater → target (belief), flipping belief to OUT
     e. Returns summary dict
5. Assert returned fields, then re-fetch belief → OUT
```

The cascade test adds a layer: `d1 → d2`. Defeating `d1` causes `d2` to lose its only justification, so `d2` also flips to OUT.

The retraction test reverses the chain: retract scope belief → defeater loses its antecedent → defeater goes OUT → target regains its justification → target back to IN.

## Invariants

1. **Scope beliefs are IN premises with no justifications.** They are atomic assertions (`scope["justifications"] == []`), not derived. They carry `metadata.scope_of` (which premise they scope) and `metadata.for_defeater` (which defeater consumes them).

2. **The defeater is always IN and derived.** It has exactly one SL justification whose antecedents are the scope-belief IDs. It carries `metadata.defeats_node` and `metadata.defeats_justification`.

3. **Scope-belief IDs are unique across defeats on the same node.** `test_scope_ids_are_unique` asserts two separate `defeat_with_scope` calls on different justification indices produce non-overlapping scope IDs.

4. **Defeat is reversible via retraction.** Retacting any scope belief must cascade the defeater to OUT and restore the target to IN.

5. **`defeat_reason_type` is optional.** When omitted (empty string), it is *not* stored in metadata. When provided, it appears in both the return dict and the defeater's metadata.

6. **Export/import preserves structural integrity.** The exported JSON must contain the scope beliefs, the defeater, its SL justification linking to scope IDs, and the target's OUT state after re-import.

## Error Handling

The file tests four explicit error paths (all in `test_defeat_with_scope_errors`):

| Condition | Exception | Message pattern |
|-----------|-----------|-----------------|
| `node_id` doesn't exist | `KeyError` | `"not found"` |
| Node exists but has no justifications | `ValueError` | `"no justifications"` |
| `just_idx` exceeds justification count | `IndexError` | `"out of range"` |
| `scope_findings` is an empty list | `ValueError` | `"scope_findings must not be empty"` |

Notably, errors are **raised as exceptions** (not returned as error dicts), meaning callers must wrap in try/except or `pytest.raises`. There is no "soft fail" path — the API is all-or-nothing.

## Topics to Explore

- [function] `reasons/api.py:defeat_with_scope` — The implementation under test; understanding its internal steps (scope-belief creation, defeater derivation, truth-flip) is essential for maintaining these tests.
- [file] `reasons/pg.py` — The proof-graph engine that resolves SL justifications and propagates truth values; the cascade and retraction tests depend on its semantics.
- [file] `tests/test_defeat_justification.py` — The simpler defeater path (no scope beliefs); comparing it to this file clarifies what scope-beliefs add structurally.
- [file] `reasons/network.py` — Cascade/retraction logic; explains *why* retracting a scope belief restores the target and *why* defeating D1 cascades to D2.
- [general] `sl-justification-semantics` — The `sl` (Set of Links) justification type is the backbone of the belief graph; understanding how antecedents, truth values, and retraction interact is a prerequisite for any of these tests.

## Beliefs

- `scope-beliefs-are-atomic-premises` — Scope beliefs created by `defeat_with_scope` are always IN with zero justifications; they are leaf premises, not derived nodes.
- `defeater-requires-scope-beliefs-as-antecedents` — The defeater node's single SL justification must list the scope-belief IDs as antecedents; a defeater without this link is structurally invalid in this codebase.
- `retracting-scope-belief-restores-target` — Retraction of any scope belief cascades through the defeater to restore the target node to IN; this is a hard invariant, not a best-effort.
- `defeat-with-scope-preserves-export-roundtrip` — The full structure (scope beliefs, defeater, SL links, target OUT state) must survive `export_network` → JSON → `import_json` without data loss.
- `defeat-reason-type-omitted-means-absent-from-metadata` — When `defeat_reason_type` is empty/unset, the key is deliberately *absent* from the defeater's metadata dict (not present as an empty string).
