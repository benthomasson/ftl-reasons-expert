---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_hide_out.py

## Purpose

`tests/test_hide_out.py` is the regression test suite for issue **#243**, which introduced the rule that *retracted ("OUT") beliefs must not be surfaced to model-facing retrieval by default*. It exercises three public API entry points — `api.search`, `api.lookup`, and `api.compact` — and validates two behaviors for each:

1. **Default (safe) behavior**: retracted beliefs are invisible.
2. **Opt-in behavior**: `include_out=True` restores the old "show everything" view for human/debug callers.

Its role is purely contract verification: it does not test storage internals, only the externally observable text produced by the retrieval layer. This is a "seam" test — if a future refactor of storage or the search pipeline accidentally re-exposes retracted nodes, this file fails.

## Key Components

### `_setup_mixed_db(tmp_path) -> str`
A reusable fixture helper that seeds a temp database with a controlled mix of statuses:

```python
api.add_node("premise-in",  "Retraction cascades propagate correctly", db_path=db)
api.add_node("premise-out", "Retraction cascades are broken",           db_path=db)
api.retract_node("premise-out", db_path=db)
api.add_node("derived-in", "System handles retraction well",
             sl="premise-in", db_path=db)
```

- Both `premise-in` and `premise-out` share the token `"retraction"`, so a single query `"retraction"` matches both statuses — the ideal setup to prove filtering works.
- `derived-in` is attached to `premise-in` via `sl=` (a support/dependency link), so search must also correctly handle a live node that has a dependency.
- Returns the DB path as a `str` (all `api` calls take `db_path` as keyword).

### `TestSearchHidesOut`
Five tests covering `api.search`:

| Test | Contract |
|---|---|
| `test_default_excludes_out` | Default query for `"retraction"` contains `premise-in` but not `premise-out`. |
| `test_include_out_shows_all` | `include_out=True` contains both. |
| `test_no_results_when_only_out_matches` | When the only match is OUT, output contains `"No results found"`. |
| `test_include_out_finds_only_out` | `include_out=True` returns `only-out` for a query that would otherwise be empty. |
| `test_out_neighbors_not_expanded` | A retracted node that is a *dependent* of a live node is not surfaced as a neighbor: search output contains `"### root"` but not `"### derived-out"`. This is the strongest guard — filtering must happen before adjacency expansion, not just on the primary hits. |

The `"### root"` marker reveals the search renderer's heading format (a Markdown `### <node_id>` heading per hit).

### `TestLookupHidesOut`
Three tests mirroring the search contract for `api.lookup` (`"No beliefs found"` is the empty-result string — note it differs from search's `"No results found"`). Lookup is tested for default exclusion and `include_out=True` inclusion, but *not* for neighbor expansion (lookup is direct-match only).

### `TestCompactHidesOut`
Two tests covering `api.compact`, which returns a full dump of the DB rather than matching a query:

- `test_default_excludes_out_section` — output contains `active`, does not contain `retracted`, and crucially does not contain the literal section header `"OUT (retracted)"`.
- `test_include_out_shows_out_section` — with `include_out=True`, the node appears *and* the `"OUT (retracted)"` header is rendered.

The literal string `"OUT (retracted)"` is a rendering contract, not just a filter contract — the whole section is omitted by default.

## Patterns

- **Pytest `tmp_path` fixture for isolation**: every test gets a fresh SQLite file; there is no shared state and no teardown needed.
- **Class-based grouping by API surface**: `TestSearchHidesOut`, `TestLookupHidesOut`, `TestCompactHidesOut` — the same three-scenario matrix (default / include_out / empty) is repeated per API, which makes coverage gaps obvious.
- **Black-box, string-assertion testing**: the tests do not inspect the DB or internal objects; they assert on substring presence/absence in rendered output. This couples the tests to output format (`"### <id>"`, `"No results found"`, `"No beliefs found"`, `"OUT (retracted)"`), which is intentional — those strings *are* the model-facing contract.
- **Private helper convention**: `_setup_mixed_db` is prefixed with `_` and module-level, keeping the fixture logic out of the test classes.
- **Sentinel tokens**: identifiers like `"premise-in"`, `"premise-out"`, `"xyzterm"` are chosen to be unambiguous substrings so a simple `in` check is a valid assertion.

## Dependencies

- **Imports**: `from reasons import api`. The repository ships both `reasons/` and `reasons_lib/` with parallel module layouts, so `reasons` is presumably a thin namespace/facade over `reasons_lib`; these tests bind to the `reasons` façade, meaning the façade must re-export `init_db`, `add_node`, `retract_node`, `search`, `lookup`, and `compact`.
- **Test infrastructure**: `pytest` (via the `tmp_path` fixture) and `tests/conftest.py` (not imported explicitly, but the project's test config lives there).
- **Depends on**: the storage layer that records retraction status, and the retrieval/rendering layer that formats results.
- **Imported by**: nothing. This is a leaf test module — no other module should import it.

## Flow

The execution pattern is identical across tests:

1. **Arrange** — `tmp_path` provides a unique directory; `_setup_mixed_db` (or inline setup) calls `api.init_db(db)` then `api.add_node(..., db_path=db)` a handful of times, using `api.retract_node(id, db_path=db)` to flip chosen nodes to OUT.
2. **Act** — a single call to one of `api.search(query, db_path=db[, include_out=True])`, `api.lookup(...)`, or `api.compact(db_path=db[, include_out=True])`.
3. **Assert** — substring checks on the returned `str`. Positive assertions confirm the surviving node text appears; negative assertions confirm the retracted node's ID/text does not; empty-result tests assert on the vendor-specific sentinel message.

No global state is mutated; each test is independent and order-insensitive.

## Invariants

- **Status gates visibility**: A node retracted via `api.retract_node` must not appear in `search`/`lookup`/`compact` output unless `include_out=True` is passed.
- **`include_out` is additive, not exclusive**: passing `include_out=True` must *add* OUT nodes without removing IN nodes (asserted by requiring both IDs in the result).
- **Filtering precedes neighborhood expansion**: `test_out_neighbors_not_expanded` requires that a retracted dependent of a live node not be rendered as an adjacent hit — implying the filter is applied to the candidate set before expansion, not to the rendered page.
- **Empty-result sentinels are API-specific**: search → `"No results found"`; lookup → `"No beliefs found"`. These strings are load-bearing and change-averse.
- **Section header contract for compact**: the `"OUT (retracted)"` heading is emitted iff `include_out=True`.
- **Rendering format for hits**: search hits are rendered with `### <node_id>` headings (otherwise the `### root` assertion would fail).
- **`include_out` defaults to `False`** across all three APIs — the tests exercise the default (no kwarg) path extensively.

## Error Handling

The file contains almost no error-path testing. Setup errors (invalid DB path, duplicate node IDs) would surface as exceptions from `api.init_db` / `api.add_node` and fail the test. The only "error-like" thing deliberately tested is the **empty-result contract**: rather than returning an empty string or raising, the retrieval APIs return a human-readable sentinel (`"No results found"`, `"No beliefs found"`) that callers (and the model) can render. There is no `pytest.raises`, no exception-message assertion, and no test of `include_out` interaction with malformed input.

## Topics to Explore

- [file] `reasons/api.py` — Where `search`, `lookup`, and `compact` are declared; check how `include_out` is threaded down to storage/rendering.
- [file] `reasons/storage.py` — How retraction status is persisted (status column? provider flag?), which the `include_out` filter must query.
- [function] `reasons_lib/api.py:search` — The implementation behind `TestSearchHidesOut`; verify the filter is applied before neighbor expansion (`test_out_neighbors_not_expanded`).
- [file] `CHANGELOG.md` — Locate the entry for issue #243 to understand the motivation for hiding OUT beliefs from model-facing retrieval.
- [general] `reasons-vs-reasons_lib-shim` — Confirm how the `reasons` package re-exports `reasons_lib`; these tests bind to the facade, so a broken re-export would break them.

## Beliefs

- `search-default-excludes-out` — `api.search` omits retracted nodes from its results when `include_out` is not specified.
- `include-out-enables-out-visibility` — Passing `include_out=True` causes all three retrieval APIs (`search`, `lookup`, `compact`) to include retracted nodes.
- `empty-result-message-is-api-specific` — `api.search` returns text containing `"No results found"` while `api.lookup` returns `"No beliefs found"` when no IN matches exist.
- `compact-renders-out-section-header-only-when-requested` — `api.compact` emits the literal header `"OUT (retracted)"` if and only if `include_out=True`.
- `out-filter-precedes-neighbor-expansion` — Retracted beliefs are excluded even when they are dependents of a live node returned by `api.search` (`### derived-out` is absent from output).
- `tests-bind-to-reasons-facade` — The suite imports `api` from `reasons`, not `reasons_lib`, so the `reasons` package must re-export all six used symbols.
