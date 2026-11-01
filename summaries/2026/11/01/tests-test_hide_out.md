---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_hide_out.py

# `tests/test_hide_out.py` — Hiding Retracted Beliefs from Retrieval

## Purpose

This file owns the **regression tests for issue #243**: ensuring that retracted (OUT) beliefs are invisible to the three primary model-facing retrieval surfaces — `search()`, `lookup()`, and `compact()` — unless the caller explicitly opts in via `include_out=True`. It protects the invariant that a belief once retracted should not pollute LLM context windows, search results, or compact summaries unless a developer deliberately requests the full history.

## Key Components

| Component | Role |
|---|---|
| `_setup_mixed_db(tmp_path)` | Shared fixture-builder. Creates a SQLite DB with one IN premise (`premise-in`), one OUT premise (`premise-out`, retracted), and one derived node (`derived-in`) that depends on the IN premise. Returns the DB path string. |
| `TestSearchHidesOut` | Five tests validating that `api.search()` filters OUT beliefs by default and includes them with `include_out=True`. Also verifies that OUT neighbors aren't expanded in the result graph. |
| `TestLookupHidesOut` | Three tests validating the same default/opt-in contract for `api.lookup()`. |
| `TestCompactHidesOut` | Two tests validating that `api.compact()` omits the `"OUT (retracted)"` section by default and renders it when `include_out=True`. |

Each test class follows the same **three-scenario contract**:

1. **Default excludes OUT** — OUT beliefs absent from results.
2. **`include_out=True` shows all** — both IN and OUT beliefs present.
3. **Only-OUT edge case** (search/lookup) — when the *sole* matching belief is OUT, the default call returns a "no results" sentinel string rather than an empty list.

## Patterns

- **Grouped test classes per API surface.** Each class isolates one retrieval function (`search`, `lookup`, `compact`) so that a regression in one surface doesn't mask a pass in another.
- **`tmp_path` isolation.** Every test gets a fresh directory via pytest's `tmp_path` fixture, so databases never leak state across tests. No shared DB, no class-scoped fixtures.
- **String-assertion style.** The API returns formatted text (markdown-like), not structured objects, so tests assert on substrings (`"premise-in" in result`, `"### derived-out" not in result`). This is a deliberate trade-off: it tests the *rendered output* a model would see, rather than internal data structures.
- **Helper DRY with inline setup.** `_setup_mixed_db` is used where the "mixed IN/OUT" scenario is needed, but the "only-OUT" edge-case tests build their own minimal DB inline because the topology is different (single node, no derivations).
- **Sentinel-string assertions.** `"No results found"` and `"No beliefs found"` are the API's zero-result messages; the tests pin these exact strings to catch accidental format changes.

## Dependencies

- **Imports:** `from reasons import api` — the sole dependency. The tests exercise `api.init_db`, `api.add_node`, `api.retract_node`, `api.search`, `api.lookup`, and `api.compact`.
- **Imported by:** Nothing imports this file. It is a leaf test module discovered by pytest.
- **Upstream context:** The `api` module is a thin façade over the belief-graph storage layer (`reasons/storage.py`, `reasons/pg.py`). The `include_out` parameter was added as part of the #243 fix, so the tests here serve as the acceptance criteria for that change.

## Flow

```
test  ──►  _setup_mixed_db(tmp_path)
                │
                ├─ api.init_db(db)
                ├─ api.add_node("premise-in", …)        # IN status
                ├─ api.add_node("premise-out", …)       # IN → …
                ├─ api.retract_node("premise-out")      # … → OUT
                └─ api.add_node("derived-in", sl="premise-in")  # derived, IN
                │
                ▼
          returns db_path string
                │
                ▼
test  ──►  api.search("retraction", db_path=db)
                │
                ├─ scans all nodes matching "retraction"
                ├─ filters out nodes with status == OUT  ← the invariant under test
                ├─ expands neighbors of matched IN nodes
                │   (OUT neighbors are NOT expanded)
                └─ renders markdown string
                │
                ▼
assert  "premise-in" in result
assert  "premise-out" not in result
```

The same pattern repeats for `lookup` and `compact`, with `compact` additionally checking for the presence/absence of the literal section header `"OUT (retracted)"`.

## Invariants

1. **Default visibility contract:** Every model-facing retrieval call (`search`, `lookup`, `compact`) must exclude OUT-status beliefs unless `include_out=True` is explicitly passed. This is the core invariant of #243.
2. **Neighbor expansion respects OUT status:** Even if an IN node is matched, its OUT children/derivations must not be rendered in the expanded output (`test_out_neighbors_not_expanded`).
3. **Zero-result sentinel is preserved:** When the *only* matching belief is OUT, the default call must still return the "No results" / "No beliefs" sentinel rather than an empty string or a list — the caller can distinguish "no data" from "all data was filtered."
4. **Opt-in is total:** `include_out=True` must surface *all* beliefs, IN and OUT, with no partial filtering.
5. **Compact section header:** The `compact()` output uses the exact section header `"OUT (retracted)"` when including OUT beliefs. Tests pin this string.

## Error Handling

This test file **does not test error paths**. All tests assume the happy path:

- No tests cover invalid `db_path`, missing nodes, or malformed queries.
- No tests assert on exceptions from `retract_node` on an already-retracted node.
- The "No results found" assertions are the closest thing to an error-ish path, but they verify the *empty-result contract*, not an exception.

If `api.search` raised instead of returning a sentinel string, `test_no_results_when_only_out_matches` would fail with an unhandled exception, which pytest would report as an error rather than a test failure. In practice, the API is designed to be total (always returns a string), so no exception handling is needed here.

---

## Topics to Explore

- [file] `reasons/api.py` — The module under test. Understand how `search`, `lookup`, and `compact` implement the `include_out` filter and how they format their string output.
- [function] `reasons/api.py:retract_node` — How a belief transitions from IN to OUT. Understanding the state mutation helps predict which other operations (e.g., `derive`, `repair`) are affected by retraction.
- [file] `tests/test_outlist.py` — A sibling test file that likely covers the OUT-list data structure or the `outlist` API surface, complementing this file's focus on retrieval visibility.
- [file] `reasons/compact.py` — The implementation behind `api.compact()`. Shows how the "OUT (retracted)" section is conditionally rendered and how belief statuses drive formatting.
- [repo] `CHANGELOG.md` — Look for the #243 entry to understand the motivation, the exact scope of the fix, and whether any follow-up issues were filed.

---

## Beliefs

- `search-lookup-compact-hide-out-by-default` — All three model-facing retrieval functions (`api.search`, `api.lookup`, `api.compact`) exclude OUT-status beliefs from their output unless the caller passes `include_out=True`.
- `out-neighbor-exclusion-in-search` — When `api.search` expands the neighborhood of a matched IN node, OUT-status children are excluded from the rendered graph even if they would otherwise appear as adjacent nodes.
- `zero-result-sentinel-survives-filtering` — When the only belief matching a query is OUT-status, `api.search` returns `"No results found"` and `api.lookup` returns `"No beliefs found"` rather than an empty string, preserving a distinguishable empty-result contract.
- `compact-out-section-header-is-literal` — The `api.compact` function renders retracted beliefs under the exact section header `"OUT (retracted)"`, and this header is absent from the output when `include_out` is falsy.
- `tmp-path-db-isolation-per-test` — Every test in this file creates its own SQLite database under pytest's `tmp_path`, ensuring no cross-test state leakage and making the test suite order-independent.
