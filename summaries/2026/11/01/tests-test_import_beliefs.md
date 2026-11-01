---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_import_beliefs.py

# `tests/test_import_beliefs.py`

## 1. Purpose

This file is the test suite for the **belief-import pipeline** — the mechanism by which a human-authored `beliefs.md` (and optional `nogoods.md`) markdown document gets loaded into the RMS (Reasoning Maintenance System) network as a graph of justified beliefs.

Its responsibility is to verify three layers of the import contract:

1. **Parsing** — that the markdown→dict extraction (`parse_beliefs`, `parse_nogoods`) is faithful.
2. **Graph construction** — that `import_into_network` correctly maps parsed dicts onto `Network` nodes, justifications, and nogoods.
3. **Semantic correctness** — that the resulting network *behaves* (cascading retraction, deduplication, stale-retraction, reverse-index integrity) as the rest of the system expects.

In short: if you change the markdown schema or the import logic in `reasons/import_beliefs.py`, this file tells you what broke and why.

---

## 2. Key Components

### Module-level fixtures (data constants)

| Constant | Shape | Role |
|----------|-------|------|
| `SAMPLE_BELIEFS` | Markdown string, 5 claims | Exercises every field: `id`, `[STATUS]`, `TYPE`, text, `Source`, `Source hash`, `Date`, `Depends on`, `Stale reason`, `Superseded by`, `Unless` |
| `SAMPLE_NOGOODS` | Markdown string, 2 nogoods | Exercises `id`, label, `Discovered`, `Discovered by`, `Resolution`, `Affects` |

These are **not** pytest fixtures — they are plain module constants so every test can reference them without setup/teardown. They act as a *specification* of the expected markdown format.

### Test classes

- **`TestParseBeliefs`** (9 tests) — White-box tests of `parse_beliefs()`. Each test asserts one field of the returned list-of-dicts. Notable: `test_parses_unless` and `test_parses_unless_empty_default` confirm the `unless` (defeater) list defaults to `[]`.

- **`TestParseNogoods`** (4 tests) — Verifies `parse_nogoods()` extracts `id`, `label`, `affects` (list), and `resolution`.

- **`TestImportIntoNetwork`** (13 tests) — The core behavioral suite. Constructs a fresh `Network()`, calls `import_into_network(net, SAMPLE_BELIEFS [, SAMPLE_NOGOODS])`, then inspects `net.nodes`, `net.nogoods`, and the return-dict counters.

- **`TestImportRealRegistry`** (1 test + 2 fixtures) — An **optional integration test** that reads `~/git/beliefs-pi/beliefs.md` from the developer's filesystem. Skips via `pytest.skip()` if the file isn't present.

### Functions under test (imported from `reasons.import_beliefs`)

- `parse_beliefs(text: str) -> list[dict]`
- `parse_nogoods(text: str) -> list[dict]`
- `import_into_network(net: Network, beliefs_md: str, nogoods_md: str = "") -> dict`

The return dict of `import_into_network` is a **counting envelope**:
```python
{
    "claims_imported": int,
    "claims_retracted": int,
    "claims_skipped": int,
    "nogoods_imported": int,
}
```

---

## 3. Patterns

- **Spec-by-example data constants.** Rather than a JSON/CSV fixture, the "schema" is embedded as the markdown strings at the top of the file. Any developer reading the test immediately sees the expected document format.

- **One-assertion-per-concern grouping.** Each test name maps to a single field or behavior (e.g. `test_parses_source` checks *only* `source` and `source_hash`). This makes failure messages unambiguous.

- **`by_id` lookup idiom.** Repeated pattern:
  ```python
  by_id = {c["id"]: c for c in claims}
  assert by_id["derived-c"]["depends_on"] == ["premise-a", "premise-b"]
  ```
  Avoids brittle index-based access when the list order might change.

- **Behavioral > structural tests in `TestImportIntoNetwork`.** Beyond checking field values, tests like `test_cascading_works_after_import` and `test_dependency_chain` verify that the *network semantics* (SL justifications, retraction propagation) are correct after import.

- **Optional integration via `pytest.skip`.** `TestImportRealRegistry` degrades gracefully: if the local `beliefs-pi` repo isn't checked out, the test is skipped rather than failing CI.

- **Idempotency / dedup verification.** `test_skip_duplicates` calls `import_into_network` twice and asserts the second pass imports 0 and skips 5 — confirming the import is idempotent.

---

## 4. Dependencies

### This file imports

| Module | Symbol | Why |
|--------|--------|-----|
| `pathlib` | `Path` | Resolving `~/git/beliefs-pi/…` in the integration test |
| `pytest` | `pytest`, `@pytest.fixture` | Test runner, skip logic, fixture injection |
| `reasons.network` | `Network` | The graph structure being populated |
| `reasons.import_beliefs` | `parse_beliefs`, `parse_nogoods`, `import_into_network` | The unit under test |

### What imports this file

Nothing. It is a leaf test module — only discovered by pytest via the `tests/` directory convention. No other module imports it.

### Implicit dependency

The `conftest.py` in `tests/` may provide shared fixtures (e.g. a temporary `Network` or database session), though this particular file doesn't reference any such fixture explicitly.

---

## 5. Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Developer / CI runs:  pytest tests/test_import_beliefs.py          │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TestParseBeliefs / TestParseNogoods                                │
│                                                                     │
│  SAMPLE_BELIEFS (str) ──► parse_beliefs() ──► list[dict]           │
│  SAMPLE_NOGOODS (str) ──► parse_nogoods() ──► list[dict]           │
│                                                                     │
│  Assertions: field-by-field comparison of dict values              │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TestImportIntoNetwork                                              │
│                                                                     │
│  Network() ──┐                                                      │
│              ├──► import_into_network(net, beliefs_md, nogoods_md)  │
│  SAMPLE_* ───┘         │                                            │
│                         ▼                                            │
│              Internal steps (implied):                               │
│              1. parse_beliefs → list[dict]                           │
│              2. For each claim:                                      │
│                 a. If id already in net.nodes → skip (count++)       │
│                 b. Create Node with truth_value from status          │
│                    (IN→"IN", STALE→"OUT")                            │
│                 c. If type==OBSERVATION → justifications=[]          │
│                 d. If type==DERIVED → create SL Justification       │
│                    with antecedents = depends_on list                │
│                 e. Set source, source_hash, date, metadata           │
│                 f. Register reverse `dependents` index               │
│              3. parse_nogoods → list[dict]                           │
│              4. For each nogood:                                     │
│                 a. If any node in `affects` not in net.nodes → skip  │
│                 b. Else create Nogood entry                          │
│              5. Return counting dict                                 │
│                         │                                            │
│                         ▼                                            │
│  Assertions on net.nodes, net.nogoods, return dict,                │
│  and cascading behavior (net.retract("premise-a"))                 │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TestImportRealRegistry  (skips if ~/git/beliefs-pi absent)        │
│  Reads real files → same import_into_network → count assertions     │
└─────────────────────────────────────────────────────────────────────┘
```

The critical data transformation is **markdown → dict → graph node with typed justifications**. The `STALE` → `OUT` mapping and `OBSERVATION` → no-justification rules are the two most important semantic mappings.

---

## 6. Invariants (rules this suite enforces)

| # | Invariant | Enforced by |
|---|-----------|-------------|
| 1 | Every `OBSERVATION`-type claim is imported as a **premise** (empty `justifications` list) | `test_imports_premises` |
| 2 | Every `DERIVED`-type claim gets exactly one **SL justification** whose `antecedents` match its `Depends on` list | `test_imports_derived_with_justification` |
| 3 | A claim marked `[STALE]` is imported with `truth_value == "OUT"` (retracted), but its `stale_reason` and `superseded_by` are preserved in `metadata` | `test_stale_claims_retracted`, `test_stale_metadata_preserved` |
| 4 | Import is **idempotent**: re-importing the same beliefs results in 0 imported, N skipped | `test_skip_duplicates` |
| 5 | The reverse `dependents` index is populated: if B depends on A, then `net.nodes[A].dependents` contains B | `test_dependents_registered` |
| 6 | Retracting a premise **cascades** through all downstream DERIVED nodes (multi-hop) | `test_cascading_works_after_import` |
| 7 | Nogoods referencing non-existent nodes are **silently skipped** (not an error) | `test_nogoods_without_valid_nodes_skipped` |
| 8 | The return dict is an accurate count: `claims_imported + claims_retracted + claims_skipped` accounts for every parsed claim | `test_import_counts` |

---

## 7. Error Handling

This is a **test** file, so error handling is minimal by design, but there are a few notable patterns:

- **`pytest.skip`** in `TestImportRealRegistry` — the `real_beliefs` fixture calls `pytest.skip("beliefs-pi not found")` when the optional file is absent. This turns a *missing precondition* into a *passive skip* rather than a collection error or test failure. The `real_nogoods` fixture returns `None` (not skip), so a missing `nogoods.md` is treated as "no nogoods to import."

- **Graceful degradation for invalid nogoods** — `test_nogoods_without_valid_nodes_skipped` confirms that `import_into_network` does **not** raise when a nogood references nodes that don't exist. The error is swallowed and reflected in the count (`nogoods_imported == 0`). This is a deliberate design choice: partial imports are acceptable.

- **No exception-path tests.** The suite doesn't test what happens with malformed markdown (e.g. a claim with no `###` header, a missing `Source` line). This is a gap — if `parse_beliefs` raises on malformed input, there's no test pinning that behavior down.

- **No mocking.** All tests use real `Network()` instances. If `Network.__init__` has side effects (file I/O, DB connection), these tests would need those to be lightweight or already handled in `conftest.py`.

---

## Topics to Explore

- [file] `reasons/import_beliefs.py` — The production implementation of `parse_beliefs`, `parse_nogoods`, and `import_into_network`; the regex/logic that this test suite validates.
- [file] `reasons/network.py` — The `Network`, `Node`, and `Justification` classes; understanding the SL justification type and the `retract()` cascade logic is essential to interpreting the import tests.
- [file] `tests/conftest.py` — Shared fixtures and any `Network` or storage setup that these tests may rely on implicitly.
- [file] `reasons/import_agent.py` — The sibling importer; comparing its test (`test_import_agent.py`) reveals whether both importers share a common contract or diverge in dedup/stale handling.
- [general] `beliefs-md-schema` — The markdown format itself (headers, bullet fields, `[STATUS] TYPE` tokens) is the de-facto API between the human-authored document and the parser; any schema evolution must update both `SAMPLE_BELIEFS` here and the parser.

---

## Beliefs

- `stale-claim-maps-to-out` — A claim with `[STALE]` status is always imported as a node with `truth_value == "OUT"`, never as `"IN"`, regardless of its `Depends on` list.
- `observation-produces-empty-justifications` — Any claim of type `OBSERVATION` is imported with an empty `justifications` list (i.e., it is a premise, not a derived conclusion).
- `import-into-network-is-idempotent` — Calling `import_into_network` a second time with the same markdown produces `claims_imported == 0` and `claims_skipped == N`, with no duplicate nodes or justifications.
- `nogood-requires-valid-nodes` — A nogood whose `Affects` list references any node ID not present in `net.nodes` is silently skipped (not imported, not raised), reducing `nogoods_imported` by one.
- `import-populates-reverse-dependents` — For every edge A→B created by a `Depends on` relationship, `net.nodes[A].dependents` must contain `B`, enabling O(1) lookup for retraction cascades.
