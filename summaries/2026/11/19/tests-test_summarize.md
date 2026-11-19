---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_summarize.py

# `tests/test_summarize.py`

## Purpose

This file is the **behavioral spec for Doyle-style SUM/GF summarization** in the `reasons` codebase. It validates the feature that lets a `summary` node stand in for a group of "covered" nodes: the summary node participates in the truth-maintenance graph (via an SL justification over the covered nodes), gets bidirectional metadata linking it to those nodes, appears in `compact()` output, and — crucially — causes the covered nodes to be *hidden* from compact output while the summary is IN.

It owns four layers of contract:

1. `Network.summarize(...)` — in-memory domain behavior.
2. `compact(net, ...)` — rendering behavior (hide/restore, informational counts).
3. `Storage` — save/load round-trip of summary metadata and truth value.
4. `api.summarize(...)` / `api.compact(...)` — the DB-backed public surface.

## Key Components

### `TestSummarizeBasic` — `Network.summarize` contract

- **`test_create_summary`** pins the exact return shape: `{"summary_id": ..., "truth_value": "IN", ...}` plus side-effect `net.nodes["summary-abc"].metadata["summarizes"] == ["a", "b", "c"]`.
- **`test_summary_depends_on_covered_nodes`** asserts the summary node's *first* justification is `type == "SL"` with `antecedents` exactly `{"a", "b"}` — i.e. the summary is supported-if-and-only-if all covered nodes are IN.
- **`test_summary_out_when_covered_node_out`** / **`test_summary_restored_when_covered_restored`** exercise the truth-maintenance cascade: `net.retract("a")` flips summary to `OUT`; `net.assert_node("a")` flips it back to `IN`. This is the classic SL semantics — one missing antecedent taints the summary.
- **`test_covered_nodes_get_metadata`** checks the reverse index: each covered node gets `metadata["summarized_by"]` containing the summary id.
- **`test_nonexistent_node_raises`** — `over=["missing"]` raises `KeyError`.
- **`test_duplicate_summary_raises`** — repeated `summarize("s", …)` raises `ValueError`.

### `TestSummarizeInCompact` — rendering contract

These are **string-substring assertions** against `compact(net, budget=5000)`, which is the entire output format contract:

- Summary node renders as `"s:"` and the word `"summary"` appears.
- Covered node text like `"a: Premise A detail"` must **not** appear.
- Uncovered nodes (`"c:"`) still appear.
- When hiding happens, the literal phrase `"hidden by summaries"` is present.
- The per-summary line contains `"covers 2 nodes"`.
- **`test_out_summary_does_not_hide`** is the sharpest invariant: retracting `"a"` makes the summary `OUT`; even though `"b"` is still `IN`, `"b:"` must appear in the output — so hiding is gated on `summary.truth_value == "IN"`, not merely on summary existence.
- **`test_no_summary_no_hiding`** guards the negative case: no summary → no `"hidden by summaries"` marker and every node id is visible.

### `TestSummarizePersistence` — storage contract

Small but important: a `Storage(db).save(net)` / `.load()` round-trip must preserve

- `loaded.nodes["s"].metadata["summarizes"]`
- `loaded.nodes["s"].truth_value == "IN"`
- `loaded.nodes["a"].metadata["summarized_by"]` contains `"s"`
- And the retraction cascade still works **after load** (`loaded.retract("a")` → summary goes `OUT`), proving justifications are also restored and re-linked.

`tmp_path` is pytest's per-test temp dir; a fresh SQLite file is created each test, so there's no cross-test state.

### `TestSummarizeAPI` — public surface

- **`test_summarize_api`** pins the API return dict keys: `summary_id`, `truth_value`, `over` (note: the API also returns `over`, unlike the raw `Network.summarize` test which only checked `summarizes` metadata).
- **`test_summarize_affects_compact`** confirms the DB-backed `api.compact(...)` respects summaries identically to the in-memory `compact(net, …)`.

## Patterns

1. **Three-tier test pyramid in one file**: domain (`Network`) → rendering (`compact`) → persistence (`Storage`) → public API (`api`). Each tier is a separate test class. When you change one layer, breakage is localized.
2. **In-memory vs. DB parity**: every behavior in `TestSummarizeBasic` is re-checked at the `api` layer where DB persistence is implied. This mirrors the repo's dual module layout (`reasons/` and `reasons_lib/`) — the tests treat `Network` as the canonical implementation and `api` as the persistence wrapper.
3. **Fixture injection via `tmp_path`**: the persistence and API classes take `tmp_path` so each test gets an isolated DB path, and `Storage(db).close()` is called manually (no context manager / autouse fixture here).
4. **Substring-based output assertions**: `compact()` returns a plain string, so the tests assert on literal markers (`"hidden by summaries"`, `"covers 2 nodes"`, `"s:"`). These are effectively an unstable-but-explicit output contract — changing the wording is a breaking change.
5. **Truth-maintenance idioms**: `retract` / `assert_node` are used as the mechanism to drive truth values rather than mutating `truth_value` directly, which validates the real cascade path rather than a shortcut.

## Dependencies

**Imports used by this file:**

| Import | Role |
|---|---|
| `pytest` | `pytest.raises`, `tmp_path` fixture |
| `reasons.Justification` | builds `Justification(type="SL", antecedents=[...])` as input to `add_node` |
| `reasons.network.Network` | the in-memory graph under test — `add_node`, `summarize`, `retract`, `assert_node`, `nodes` |
| `reasons.storage.Storage` | `save` / `load` / `close` persistence round-trip |
| `reasons.compact.compact` | pure function `compact(net, budget=N) -> str` |
| `reasons.api` | module-level functions `init_db`, `add_node`, `summarize`, `compact` (all `db_path=`-taking) |

**What depends on this file (implicitly):**

- The CI workflow (`.github/workflows/*`) presumably runs `pytest`, so this file is a **guardrail** on `Network.summarize`, `compact`, `Storage` schema, and `api.summarize`. Breaks here mean breaking the public summarization feature.

**Note on import path**: the tests import from `reasons.*`, not `reasons_lib.*`. Given the mirrored directory tree, `reasons` is the package under test — likely the canonical one, with `reasons_lib` being the installable name or a vendored copy. This file does not exercise `reasons_lib`.

## Flow

A representative flow — `test_summary_out_when_covered_node_out`:

1. `net = Network()` — fresh in-memory graph.
2. `net.add_node("a", "Premise A")`, `net.add_node("b", "Premise B")` — nodes default to IN (premises).
3. `net.summarize("s", "Summary", over=["a", "b"])` — creates node `"s"` with an `SL` justification `[a, b]`, marks `a`/`b` as `summarized_by: ["s"]`, sets `summarizes: ["a", "b"]` on `s`, evaluates `s.truth_value = IN`.
4. `net.retract("a")` — sets `a` OUT, then runs truth-maintenance; `s`'s SL justification fails → `s` is OUT.
5. Assertions read `net.nodes["s"].truth_value`.

For compact flow (`test_summary_replaces_covered_in_compact`):

1. Build net with `a`, `b`, `c` (all premises) + `summarize("s", ..., over=["a","b"])`.
2. `compact(net, budget=5000)` walks the graph; sees `s` is a summary whose `truth_value == "IN"` → renders `s:` line, suppresses `a:` and `b:` lines, and appends `hidden by summaries` / `covers 2 nodes` lines.
3. `c` (uncovered, IN) renders normally.

For API flow (`test_summarize_affects_compact`):

1. `api.init_db(db_path=db)` — creates/initializes the SQLite store.
2. `api.add_node(...)` ×2 — writes premises to DB.
3. `api.summarize(...)` — persists summary and metadata.
4. `api.compact(budget=5000, db_path=db)` — reloads from DB, renders the same string as the in-memory path.

## Invariants

1. **SL structure is fixed**: a summary always has exactly one justification of `type == "SL"` whose antecedents are precisely the `over` set (test 2). Changing the justification type is a breaking change.
2. **Truth is derived, not stored independently**: summary `truth_value` is a pure function of the covered nodes' truth values. `retract`ing any covered node flips summary OUT; `assert_node`ing all covered nodes flips it back IN.
3. **`over` nodes must already exist**: unknown ids raise `KeyError`, and the summarize call must not partially succeed.
4. **Summary IDs are unique**: re-summarizing an existing id raises `ValueError`.
5. **Metadata is bidirectional**: `summary.metadata["summarizes"]` and every `covered.metadata["summarized_by"]` reference each other, and this survives `Storage` round-trip.
6. **Hiding is conditional on summary truth**: covered nodes are hidden in `compact()` **iff** the covering summary is `IN`. An OUT summary leaves its covered nodes visible.
7. **Output vocabulary is stable**: literal strings `"hidden by summaries"`, `"covers N nodes"`, and the `"<id>:"` node prefix are contract — modifying `compact()`'s phrasing will break these tests even if the behavior is correct.

## Error Handling

This file defines **two explicit error contracts** and otherwise expects no raising:

- **`KeyError`** — `Network.summarize` must raise if any id in `over` is not present in `net.nodes`. This is a precondition error, not a soft failure; partial state (summary node created, metadata half-written) must not leak. The test doesn't assert on state afterward, so a defensive implementation that validates first is expected.
- **`ValueError`** — duplicate summary ID. Same style: precondition violation, hard failure.

Everything else (retraction, assertion, `compact` with no matching summaries, empty `summarized_by`) is expected to be a **silent no-op or benign case** rather than an exception — e.g. `compact` simply omits `"hidden by summaries"` when nothing is hidden, and an OUT summary is not an error condition.

There's no try/except in this file; failure modes are propagated straight to pytest. That means a regression in `Network.summarize` will surface as either an unexpected exception (test crash) or a failed assertion (silent behavior drift), both with readable messages at the call site.

## Topics to Explore

- [file] `reasons/network.py` — Implements `Network.summarize`, `retract`, `assert_node`, and the truth-maintenance cascade these tests depend on
- [file] `reasons/compact.py` — Owns the `"hidden by summaries"` / `"covers N nodes"` output contract and the IN-gated hiding logic
- [file] `reasons/storage.py` — Defines the SQLite schema for node metadata and justification persistence that the round-trip test exercises
- [function] `reasons/api.py:summarize` — The DB-backed wrapper; note the return dict includes `over` (which the raw `Network.summarize` test doesn't check)
- [general] `doyle-sum-gf-justifications` — Background theory: why SUM/GF justifications model abstraction and what semantics distinguish them from generic SL

## Beliefs

- `summary-node-uses-sl-justification` — A summary node is always given exactly one `SL` justification whose antecedents are the `over` set.
- `summary-truth-cascades-from-covered-nodes` — Retracting any covered node flips the summary to `OUT`; asserting it back restores the summary to `IN`.
- `compact-hides-covered-only-when-summary-in` — Covered nodes are hidden from `compact()` output only while the covering summary's `truth_value == "IN"`; an OUT summary leaves them visible.
- `summarize-metadata-is-bidirectional-and-persisted` — Summary nodes carry `metadata["summarizes"]` and covered nodes carry `metadata["summarized_by"]`; both survive `Storage.save`/`load`.
- `summarize-validates-preconditions` — `Network.summarize` raises `KeyError` on unknown covered ids and `ValueError` on duplicate summary ids.
