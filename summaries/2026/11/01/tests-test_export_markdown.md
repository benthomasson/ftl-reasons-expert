---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_export_markdown.py

# `tests/test_export_markdown.py`

## Purpose

This file is the test suite for the `export_markdown` function — the serializer that turns a `Network` (the project's core belief graph) into a human-readable Markdown document. Its responsibility is to lock down the **exact textual contract** of that output: section headers, node labels, metadata rendering, ordering rules, and the distinction between semantic outlist entries (regular blockers) and defeaters (typed rebuttals).

If you're maintaining `reasons/export_markdown.py`, these tests are your specification. If you're changing the network model (e.g., adding a new node status or a new justification field), you update both the exporter and the corresponding assertions here.

## Key Components

| Component | Contract |
|-----------|----------|
| `TestExportMarkdown` | Plain class (no `unittest.TestCase` inheritance); methods are discovered by pytest via naming convention. No shared state — every test builds its own `Network`. |
| `test_empty_network` | Baseline: an empty `Network` still produces the document skeleton (`# Belief Registry`, `## Claims`). Guards against the exporter crashing or omitting headers on zero-node input. |
| `test_premise` | A node with `source`, `source_hash`, and `date` fields renders as `### <id> [IN] OBSERVATION` with bullet-list metadata lines. |
| `test_derived_node` | A node justified by `Justification(type="SL", antecedents=["a"])` renders as `[IN] DERIVED` with `- Depends on: a`. |
| `test_retracted_node` | After `net.retract("a")`, the node label flips to `[OUT]`. |
| `test_stale_metadata` | When metadata contains `stale_reason` and the node is retracted, the header becomes `[STALE]` (not `[OUT]`), and the stale fields are emitted as bullets. |
| `test_in_before_out` | **Ordering invariant**: all IN nodes must appear before any OUT node in the rendered document. Verified via `md.index("### a") < md.index("### b")`. |
| `test_nogoods_section` | Nogoods get their own `## Nogoods` section with a deterministic ID like `nogood-a-b`. |
| `test_beliefs_type_preserved` | A `beliefs_type` metadata value (e.g. `"WARNING"`) overrides the default `OBSERVATION` label in the header. |
| `test_multiple_antecedents` | Multiple antecedents in a single justification are comma-joined: `- Depends on: a, b`. |
| `test_defeater_in_outlist` | An outlist entry that is *also* a registered defeater node (has `defeater_type` + `defeats_node` metadata) renders as `- Defeated by: <id> (<type>)`, **not** as `- Unless: <id>`. Also asserts the word "Unless" is absent. |
| `test_mixed_outlist_defeater_and_regular` | When the same outlist contains both a defeater and a regular blocker, they render in their respective formats: `- Defeated by:` vs. `- Unless:`. |
| `test_defeater_with_reason_type` | When `defeat_reason_type` is present alongside `defeater_type`, both appear: `- Defeated by: <id> (<defeater_type>, <reason_type>)`. |

## Patterns

- **Arrange-Act-Assert (AAA)**: Every test follows the same three-phase structure — build the network, call `export_markdown`, assert on the string. No shared fixtures, no setup/teardown.
- **Substring assertions**: The tests assert on exact substrings rather than full-document equality. This is deliberate — it lets you add new sections or reorder unrelated content without breaking these tests, while still pinning the critical phrases.
- **Self-contained data**: Each test constructs a minimal network from scratch. There's no dependency on `conftest.py` fixtures (no `@pytest.fixture` decorators, no `self.net`). This keeps failures easy to reproduce.
- **Feature-grouped test names**: Names read like a mini spec (`test_defeater_in_outlist`, `test_mixed_outlist_defeater_and_regular`), making it easy to grep for a behavior.
- **No parameterization**: Despite testing similar shapes (e.g., the three defeater variants), the tests are written as separate methods. This keeps each test's intent explicit in its name at the cost of some repetition.

## Dependencies

**Imports (what this file depends on):**

- `reasons.Justification` — the justification dataclass (type, antecedents, outlist)
- `reasons.network.Network` — the belief graph; provides `add_node`, `retract`, `add_nogood`, `recompute_all`
- `reasons.export_markdown.export_markdown` — the function under test

**Imported by (what depends on this file):**

Nothing imports this test file at runtime. It is discovered and executed by pytest. The only "consumer" is the CI pipeline (`.github/workflows/`).

## Flow

Each test follows an identical control flow:

```
1. net = Network()                         # empty graph
2. net.add_node(id, label, ...)            # insert nodes (0–3 per test)
3. net.retract(id)  /  net.recompute_all()  # mutate state (optional)
4. md = export_markdown(net)               # serialize → str
5. assert "<expected substring>" in md     # verify contract
```

There is no I/O, no mocking, no async. The `Network` is an in-memory object; `export_markdown` is a pure function of that object. The data transformation is: **graph (nodes + edges + metadata) → linear Markdown string with fixed section ordering (Claims → Nogoods) and fixed per-node formatting**.

## Invariants

The tests collectively enforce these rules:

1. **Section ordering**: `# Belief Registry` → `## Claims` → (node entries) → `## Nogoods` (if any).
2. **Status precedence**: A retracted node with `stale_reason` metadata gets `[STALE]`, which takes priority over the plain `[OUT]` status.
3. **IN-before-OUT**: Within the Claims section, all nodes in IN state must precede all nodes in OUT/STALE state.
4. **Type label resolution**: `beliefs_type` metadata > justification-derived type (`DERIVED`) > default (`OBSERVATION`).
5. **Defeater vs. blocker disambiguation**: An outlist entry is rendered as `- Defeated by:` only if the referenced node carries `defeater_type` metadata; otherwise it's `- Unless:`.
6. **Deterministic nogood IDs**: Formed as `nogood-<id1>-<id2>` (sorted? — the test only uses `["a","b"]`; the exporter likely sorts).

## Error Handling

This is a test file, so "error handling" means *what failure modes the tests expect*:

- **No exceptions are expected** from `export_markdown` in any of these scenarios. All tests call it bare (no `pytest.raises`), meaning the contract is: *given a valid `Network`, always return a string*.
- **No edge-case error tests exist here**: e.g., a justification referencing a non-existent antecedent, a self-referential defeater, or a node with both `stale_reason` and `beliefs_type` metadata. If the exporter raises on those, those failures live elsewhere (or are untested — a gap).
- **Assertion failures** are the only "errors" surfaced: a missing substring means the exporter changed its format. The developer debugging will see the full `md` via pytest's assertion introspection (though only the specific `assert` line is shown, not the whole document — consider adding `pytest.approx` or a helper that dumps `md` on failure).

---

## Topics to Explore

- [file] `reasons/export_markdown.py` — The function under test; read the actual serialization logic to understand how status labels, metadata bullets, and defeater-vs-outlist disambiguation are implemented.
- [file] `reasons/network.py` — The `Network` class; specifically `add_node`, `retract`, `recompute_all`, and how `outlist` on a `Justification` interacts with defeater metadata to determine final node status.
- [function] `reasons/export_markdown.py:export_markdown` — Trace the ordering logic that guarantees IN-before-OUT and the section-emission order (Claims → Nogoods).
- [file] `tests/test_export_card.py` — The sibling exporter test; compare to see if `export_card` shares the same invariants or diverges (e.g., does it also handle defeaters?).
- [general] `defeater-semantics` — How the project distinguishes a "defeater" (typed, targeted rebuttal) from a regular outlist "blocker" (generic suspension). The export tests encode this distinction, but the full semantic model lives in `reasons/contradictions.py` and `reasons/verify.py`.

---

## Beliefs

- `stale-metadata-overrides-out-status` — A retracted node whose metadata contains `stale_reason` is labeled `[STALE]` in the export, not `[OUT]`; the stale fields (`stale_reason`, `superseded_by`) are rendered as additional bullet lines.
- `defeater-vs-outlist-disambiguation` — An outlist entry is rendered as `- Defeated by: <id> (<type>)` if and only if the referenced node carries `defeater_type` metadata; otherwise it renders as `- Unless: <id>`.
- `in-nodes-precede-out-nodes-in-export` — Within the `## Claims` section, all `[IN]` node entries must appear at a lower character offset than any `[OUT]` or `[STALE]` entry.
- `beliefs-type-metadata-overrides-default-label` — If a node's metadata contains `beliefs_type`, that string replaces the default type label (`OBSERVATION` or `DERIVED`) in the `### <id> [<status>] <TYPE>` header.
- `export-markdown-is-pure` — `export_markdown(net)` performs no I/O, no mutation of `net`, and is fully determined by the network's node set, justifications, metadata, and nogood list.
