---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_import_beliefs.py

# `tests/test_import_beliefs.py` — Explanation

## Purpose

This file is the test suite for the **belief-registry import pipeline**: the mechanism that ingests externally-authored `beliefs.md` / `nogoods.md` markdown files into the runtime `Network` object (RMS = the reasoning engine). It does *not* test the markdown writer or the reasoning logic itself; it tests only the three functions exported by `reasons/import_beliefs.py`:

- `parse_beliefs(text)` — markdown → list of claim dicts
- `parse_nogoods(text)` — markdown → list of nogood dicts
- `import_into_network(net, beliefs_text, nogoods_text=None)` — mutates a `Network` and returns a counter dict

It also contains one opportunistic integration test (`TestImportRealRegistry`) that runs against a real `~/git/beliefs-pi/beliefs.md` if present, and skips otherwise.

The file covers the **round-trip contract**: whatever structure is serialized into a `beliefs.md`, parsing + importing must reconstruct the same graph state (truth values, justifications, reverse dependency index, source provenance).

## Key Components

### Module-level fixtures (markdown samples)

| Constant | Shape |
|---|---|
| `SAMPLE_BELIEFS` | `# Belief Registry` → `## Repos` (id→path map) → `## Claims` with 5 `### <id> [<STATUS>] <TYPE>` blocks |
| `SAMPLE_NOGOODS` | `# Nogoods` with 2 `### <id>: <label>` blocks, each with `Discovered`, `Discovered by`, `Resolution`, `Affects` bullets |

Claim blocks carry optional bullets `- Source:`, `- Source hash:`, `- Date:`, `- Depends on:`, `- Unless:`, `- Stale reason:`, `- Superseded by:`. The parser is expected to accept these as key/value lines following a claim header.

### Test classes

**`TestParseBeliefs`** — unit tests for `parse_beliefs`. Their common idiom is `by_id = {c["id"]: c for c in claims}` then per-key assertions. Contract verified:
- Returns exactly one dict per `###` header block (5 for the sample).
- Dict keys include `id`, `status`, `type`, `text`, `source`, `source_hash`, `depends_on` (list), `unless` (list), `stale_reason`, `superseded_by`.
- `depends_on` / `unless` default to `[]` (verified via `test_parses_unless_empty_default` and `test_parses_depends_on` which asserts `premise-a["depends_on"] == []`).
- `test_parses_unless` shows the parser accepts a standalone snippet with *no* `# Belief Registry` / `## Claims` section headers — parsing is triggered structurally by `###` headers, not by section nesting.
- The `Unless:` value is comma-split into a list.

**`TestParseNogoods`** — unit tests for `parse_nogoods`. Contract verified:
- Dict keys include `id`, `label` (text after `: `), `resolution`, `affects` (comma-split list).
- Two nogoods for two blocks.

**`TestImportIntoNetwork`** — the bulk of the test surface. Each test constructs a fresh `Network()`, calls `import_into_network(...)`, and inspects `net.nodes[...]` / `net.nogoods[...]` / the returned result dict.

Behavioral contracts verified against `Network` node attributes (`truth_value`, `justifications`, `metadata`, `source`, `source_hash`, `date`, `dependents`) and justification attributes (`type`, `antecedents`):

| Test | Contract |
|---|---|
| `test_imports_premises` | `OBSERVATION` nodes get `truth_value == "IN"` and `justifications == []`. |
| `test_imports_derived_with_justification` | `DERIVED` nodes get exactly one justification of `type == "SL"` and `antecedents` equal to the `Depends on:` list. |
| `test_stale_claims_retracted` | `[STALE]` → `truth_value == "OUT"`. |
| `test_stale_metadata_preserved` | `stale_reason` / `superseded_by` land in `node.metadata[...]`, *not* as top-level attributes. |
| `test_dependency_chain` | `chain-e.justifications[0].antecedents == ["derived-c"]` — ordering is list, not set. |
| `test_dependents_registered` | Reverse index `premise-a.dependents` contains `"derived-c"`. |
| `test_cascading_works_after_import` | After import, `net.retract("premise-a")` drives `derived-c` and `chain-e` to `OUT`. |
| `test_retraction_cascades_from_stale` | `derived-c` stays `IN` because its antecedents are `premise-a`,`premise-b` (both `IN`), not `stale-d`. |
| `test_import_counts` | First import: `claims_imported == 5`, `claims_retracted == 1`, `claims_skipped == 0`. Note: `claims_retracted` is a subset of `claims_imported` — retracted claims still count as imported. |
| `test_skip_duplicates` | Second import: `claims_imported == 0`, `claims_skipped == 5`. Idempotent by node-existence, not by content comparison. |
| `test_source_preserved` | `source`/`source_hash`/`date` are promoted to **top-level node attributes**, not `metadata`. |
| `test_import_nogoods` | `net.nogoods[0].id == "nogood-001"`, `net.nogoods[0].nodes == ["premise-a","premise-b"]` (the `Affects:` list becomes `.nodes`). Returns `nogoods_imported == 2`. |
| `test_nogoods_without_valid_nodes_skipped` | Nogoods whose `Affects:` reference IDs absent from `net.nodes` are silently skipped (`nogoods_imported == 0`). |

**`TestImportRealRegistry`** — integration test:
- `real_beliefs` fixture reads `~/git/beliefs-pi/beliefs.md`; skips if absent.
- `real_nogoods` fixture returns `None` (not skip) if `~/git/beliefs-pi/nogoods.md` is absent — so the belief file alone still exercises the importer.
- Asserts `claims_imported >= 1`, `len(net.nodes) >= claims_imported`, and `in_count > out_count` (majority of a real registry is `IN`).

## Patterns

1. **Class-grouped tests by unit under test** — one class per exported function, plus an integration class. This directly mirrors the module's public API.
2. **Fixture-as-text** — markdown samples are module-level triple-quoted strings, not external files, keeping tests self-contained and diffable. The `test_parses_unless` snippet shows the parser can be tested with the smallest possible input (no surrounding sections).
3. **Dict-comprehension lookup** (`by_id = {c["id"]: c for c in claims}`) — avoids positional indexing, so adding claims to the sample doesn't shift assertions.
4. **Structural assertions via sets** where order doesn't matter (`set(node.justifications[0].antecedents) == {"premise-a","premise-b"}`), but **list assertions** where order *does* matter (`justifications[0].antecedents == ["derived-c"]`, `nogood.nodes == ["premise-a","premise-b"]`, `depends_on == [...]`).
5. **Idempotency test** (`test_skip_duplicates`) — imports twice into the same `Network` and asserts counter transitions. This is the canonical pattern for "is the importer safe to re-run?"
6. **Cascade-after-import** (`test_cascading_works_after_import`) — a "post-condition" test: it verifies not only that the graph is built, but that the *resulting* graph behaves correctly under the network's own `retract` operation. This catches inconsistencies that individual attribute checks miss (e.g., missing reverse index edges would break cascading).
7. **Graceful skip** in integration test (`pytest.skip(...)`) rather than `xfail`/`skipif` decorator — the skip decision depends on runtime filesystem state, not static config.
8. **`reasons` vs `reasons_lib`** — the repo ships both packages. The test imports from `reasons` (the top-level app package), so any change to `reasons_lib` alone will not be caught by this file — a maintenance hazard worth understanding.

## Dependencies

**Depends on:**
- `pathlib.Path` — only for the real-registry fixture.
- `pytest` — `pytest.skip` and `@pytest.fixture`.
- `reasons.network.Network` — the graph container under test (`nodes`, `nogoods`, `retract`).
- `reasons.import_beliefs.{parse_beliefs, parse_nogoods, import_into_network}` — the unit under test.

**Depended on by:**
- Nothing directly — it's a leaf test module. It is *collected* by pytest via `tests/__init__.py` / `conftest.py`.

**Coupling hazard:** The test encodes specific field names on `Network` nodes and justification objects (`truth_value`, `justifications`, `.type`, `.antecedents`, `.dependents`, `.metadata`, `.source`, `.source_hash`, `.date`, and `net.nogoods[i].id` / `.nodes`). Any refactor of those attribute names requires updating this file, even if the importer logic itself is unchanged.

## Flow

For the parser tests, the flow is pure: a string constant → `parse_beliefs` / `parse_nogoods` → assertion on the returned list of dicts. No `Network` is involved.

For the importer tests, the canonical flow is:

1. `net = Network()` — fresh, empty.
2. `result = import_into_network(net, SAMPLE_BELIEFS[, SAMPLE_NOGOODS])`.
3. `import_into_network` internally:
   - Calls `parse_beliefs(text)`.
   - For each claim dict, checks whether `id` already exists in `net.nodes`:
     - If present → increment `claims_skipped`, do not overwrite.
     - Else → create the node, set `truth_value` from status (`[IN]` → `"IN"`, `[STALE]` → `"OUT"`), attach source provenance, copy `stale_reason`/`superseded_by` into `node.metadata`, and:
       - If `type == "OBSERVATION"` → no justification (empty list).
       - If `type == "DERIVED"` → append one justification with `type="SL"` and `antecedents = depends_on`.
     - Register reverse index: for each antecedent, add the new node's id to `antecedent.dependents`.
   - If `nogoods_text` was given, parse it and for each nogood, verify every `Affects:` id exists in `net.nodes`; only then append a nogood object to `net.nogoods` (with `.nodes == affects`) and increment `nogoods_imported`.
4. Return dict with `claims_imported`, `claims_retracted`, `claims_skipped`, `nogoods_imported`.

The cascade test then exercises a *distinct* code path — `net.retract("premise-a")` — to prove the imported graph is internally consistent with the network's inference engine.

## Invariants

- **Status→truth-value mapping is total for the sampled statuses:** `[IN]` → `"IN"`, `[STALE]` → `"OUT"`. The test suite does not exercise other statuses (`OUT`, `UNKNOWN`, etc.), so the mapping is only verified for these two.
- **`claims_retracted ⊆ claims_imported`:** the `test_import_counts` assertion (`=5` and `=1`) proves a STALE claim counts as both imported *and* retracted. Any implementation that filtered retracted claims out of `claims_imported` would fail that test.
- **Idempotency key is node ID:** re-importing the same text with the same IDs produces `claims_imported == 0`, `claims_skipped == 5`. IDs are never overwritten with new content on re-import (verified indirectly by the count, not by asserting node content is preserved — a gap).
- **OBSERVATION ⇒ no justifications; DERIVED ⇒ exactly one `SL` justification.** The derived justification's `antecedents` matches `depends_on` verbatim.
- **List-valued parser fields default to `[]`.** `depends_on` and `unless` are always lists, never `None` and never absent from the dict.
- **Reverse index consistency:** every `Depends on:` edge must also appear as a `dependents` entry on the antecedent. This is what makes `test_cascading_works_after_import` pass.
- **Provenance fields live on the node, not in `metadata`:** `source`, `source_hash`, `date` are node attributes; only `stale_reason` and `superseded_by` go into `node.metadata`. The two namespaces are deliberately separate.
- **Nogood validation is all-or-nothing:** a nogood referencing even one missing node is dropped entirely, not partially.

## Error Handling

- **`import_into_network` does not raise on missing nogood refs** — `test_nogoods_without_valid_nodes_skipped` proves the function silently skips and reports `nogoods_imported == 0`. No exception is expected or asserted.
- **Missing real-registry files produce a skip, not a failure** — the fixture calls `pytest.skip(...)` when `~/git/beliefs-pi/beliefs.md` is absent, and returns `None` (not a skip) when only `nogoods.md` is absent, so the belief file can still be imported alone.
- **What's *not* covered:** there is no test that `import_into_network` handles a malformed `###` header, an unparseable `Date:`, or a `Depends on:` referencing an undefined claim. Notably `test_parses_unless` uses `fact-a` which is never defined — proving the *parser* is lenient, but leaving open whether the *importer* validates `Depends on:` references (unlike `Affects:`, which is validated). A maintainer should treat reference validation for `Depends on:` as unspecified behavior.

---

## Topics to Explore

- [file] `reasons/import_beliefs.py` — The implementation under test: `parse_beliefs`, `parse_nogoods`, `import_into_network`, and the exact regex/key/value parsing rules the tests pin down.
- [function] `reasons/import_beliefs.py:import_into_network` — Where dedup-by-ID, status→truth-value mapping, reverse-index registration, and nogood reference validation actually live.
- [file] `reasons/network.py` — Defines the `Network` container, the node/justification attributes (`truth_value`, `justifications`, `dependents`, `metadata`), and the cascade logic invoked by `Network.retract`.
- [file] `tests/test_import_agent.py` — Sibling import test covering the agent-derived import path; useful for seeing whether the two importers share a contract or diverge.
- [general] `status-to-truth-value-mapping` — The test suite only exercises `[IN]` and `[STALE]`. Check the parser/importer for handling of `OUT`, `UNKNOWN`, or other statuses before assuming the mapping is complete.

## Beliefs

- `stale-status-maps-to-out` — Claims with `[STALE]` status are imported with `truth_value == "OUT"` while their `stale_reason` and `superseded_by` are stored under `node.metadata`.
- `derived-claims-get-sl-justifications` — Every `DERIVED` claim is imported with exactly one justification of `type == "SL"` whose `antecedents` equal the `Depends on:` list.
- `premises-have-no-justifications` — `OBSERVATION` claims are imported with `justifications == []`.
- `import-is-idempotent-by-node-existence` — Re-importing claims whose IDs already exist reports `claims_imported == 0` and `claims_skipped == N`, leaving node content untouched.
- `import-preserves-network-cascade-consistency` — After import, `Network.retract` on a premise correctly cascades to all transitive dependents (verified by `test_cascading_works_after_import`), which requires the importer to maintain both forward edges and the reverse `dependents` index.
- `nogoods-require-all-affected-nodes-to-exist` — A nogood is skipped (not partially imported) if any ID in its `Affects:` list is absent from `net.nodes`, and it contributes nothing to `nogoods_imported`.
