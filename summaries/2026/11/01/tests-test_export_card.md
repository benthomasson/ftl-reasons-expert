---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_export_card.py

# `tests/test_export_card.py`

## Purpose

This file validates the **HuggingFace EEM (Expert Ecosystem Model) card exporter** — the function that converts an in-memory `Network` (a defeasible-logic belief network) into a publishable card document. A card is a single Markdown file with YAML frontmatter that conforms to the HuggingFace model-card convention, making a reason network "shareable" as a unit of expertise.

The test's responsibility is to pin down the **output contract** of `export_card`: every frontmatter key, its default value, the belief-count arithmetic, the derivation-depth calculation, and the structural sections of the Markdown body. It does **not** test the network logic itself (truth values, justifications, nogood resolution) — only the *serialization* of an already-built network into a card.

## Key Components

### Helper: `_parse_frontmatter(text)`
Splits the card on `---` (max 2 splits) and YAML-parses the middle chunk. Contract:
- Raises `AssertionError` if the text doesn't have at least two `---` delimiters.
- Returns a `dict` (whatever `yaml.safe_load` produces from the frontmatter block).

### Helper: `_make_network(nodes, project_name, nogoods)`
A **test factory** that builds a minimal `Network` without touching disk or any service. It:
1. Creates `Network()` and sets `net.meta = {"project_name": ...}`.
2. For each entry in `nodes` (a dict of `id → spec`), instantiates a `Node`, sets `truth_value` (default `"IN"`), and attaches any `Justification` objects listed under `"justifications"`.
3. For each dict in `nogoods`, instantiates a `Nogood(**dict)` and appends to `net.nogoods`.
4. Returns the populated `Network`.

> **Note:** `Node` and `Nogood` are imported *locally* inside the function rather than at module top. This is a defensive idiom — it keeps the module importable even if those symbols aren't yet resolved at collection time, and signals they are secondary to the test's primary subject (`export_card`).

### `TestFrontmatter`
Asserts every top-level YAML key that `export_card` must emit:

| Key | Expected value / default |
|-----|--------------------------|
| `schema_version` | equals `reasons.metadata.SCHEMA_VERSION` (imported constant, not a literal) |
| `type` | `"eem"` |
| `project_name` | whatever was set in `net.meta` |
| `domain` | list passed to `export_card(domain=[…])`, or `[]` if omitted |
| `license` | passed value, or `"mit"` by default |
| `base_network` | passed string, or `None` if omitted |
| `source_repos` | list of repo slugs |
| `generator` | a string containing `"ftl-reasons/"` (asserts provenance, not exact value) |

### `TestBeliefCounts`
Verifies the arithmetic in the frontmatter:
- `beliefs_total` = number of nodes
- `beliefs_in` / `beliefs_out` = partition by `truth_value`
- `premises` = nodes with **no** justifications
- `derived` = nodes with **at least one** justification
- `nogoods` = length of `net.nogoods`

### `TestEmptyNetwork`
Edge case: a `Network` with zero nodes and zero nogoods must still produce a valid card (all counts 0, title still rendered from `project_name`).

### `TestMaxDepth`
Checks the **longest derivation chain** reported in the Markdown body:
- 0 → only premises (no edges)
- 1 → one justification pointing at a premise
- 2 → a chain `a → b → c` (two hops)

The assertion is a substring match on a Markdown table row: `"| Max derivation depth | N |"`.

### `TestMarkdownBody`
Validates the **human-readable** sections:
- Stats table rows (`| Total beliefs | 1 |`, `| Premises (observations) | 1 |`)
- Retraction rate formatting (`50%`)
- Presence of `## How to Use` with a concrete CLI example (`reasons import-json network.json`)
- `## Quality` section with a summary line
- `## License` section echoing the license string
- Title: `project_name` is title-cased into an `# H1` heading (`k8s-ops-expert` → `# K8S Ops Expert`)

## Patterns

- **Class-per-concern grouping.** Each `Test*` class isolates one aspect of the output contract. This keeps failure triage trivial — a red `TestMaxDepth` tells you the depth algorithm is broken, not that the license field is wrong.
- **Factory over fixture.** `_make_network` is a plain function (not a pytest fixture), so tests that need *different* networks call it with different args. There's no `conftest.py` coupling here.
- **Constant pinning via import.** `SCHEMA_VERSION` is imported from `reasons.metadata` rather than hardcoded, so a schema bump in the library automatically updates the expected value in tests (or fails loudly if the export lags).
- **Substring assertions for Markdown body.** The body is free-form Markdown; the tests use `"..." in card` rather than parsing the whole document. This is deliberate — it validates *presence* of required content without over-constraining formatting.
- **Local imports for secondary types.** `Node` and `Nogood` are imported inside `_make_network` to keep the test module's top-level import surface minimal and to defer any potential circularity.

## Dependencies

### Imports (what this file needs)

| Import | Role |
|--------|------|
| `yaml` | Parse frontmatter in `_parse_frontmatter` |
| `reasons.Justification` | Build justification objects in the factory |
| `reasons.metadata.SCHEMA_VERSION` | Pin the expected schema version constant |
| `reasons.network.Network` | The object under test (input to `export_card`) |
| `reasons.export_card.export_card` | **The function under test** |
| `reasons.Node` (local) | Create belief nodes in the factory |
| `reasons.Nogood` (local) | Create nogood records in the factory |

### Importers (what depends on this file)

Nothing imports this test module at runtime. It is consumed by the **test runner** (pytest). It sits alongside `tests/conftest.py`, which may provide shared fixtures, but this file is self-contained.

## Flow

```
test method
  │
  ├─► _make_network(...)          # build a small in-memory Network
  │     ├─ Network()              # fresh instance
  │     ├─ net.meta = {...}
  │     ├─ for each node spec:    # instantiate Node + Justifications
  │     └─ for each nogood spec:  # instantiate Nogood
  │
  ├─► export_card(net, **opts)    # the function under test
  │     ├─ compute belief counts (IN/OUT/premises/derived)
  │     ├─ compute max derivation depth (BFS/DFS over justification graph)
  │     ├─ render YAML frontmatter dict
  │     ├─ render Markdown body (stats table, sections)
  │     └─ return: str            # "---\n{yaml}\n---\n\n# Title\n\n..."
  │
  ├─► _parse_frontmatter(card)    # split on "---", yaml.safe_load middle
  │
  └─► assert on fm["key"] or "substring" in card
```

There is no async, no I/O, no mocking — the entire test is a **pure function → string → parse → assert** pipeline. This makes the suite fast and deterministic.

## Invariants

The tests collectively enforce these invariants on `export_card`:

1. **Frontmatter delimiters are mandatory.** `_parse_frontmatter` will `assert len(parts) >= 3`; a card missing `---` boundaries fails immediately.
2. **`type` is always `"eem"`.** Non-negotiable taxonomy marker.
3. **Defaults are explicit:** `license → "mit"`, `domain → []`, `base_network → None`. Callers can omit them safely.
4. **Belief counts are exhaustive:** `beliefs_in + beliefs_out == beliefs_total` (implicitly, via the mixed test).
5. **Premise/derived partition is exclusive:** a node either has justifications (derived) or doesn't (premise). The sum `premises + derived` equals `beliefs_total`.
6. **Max depth ≥ 0** and equals the longest justification chain length (number of edges).
7. **Title is derived from `project_name`** via `.title()` (evidenced by `k8s-ops-expert` → `K8S Ops Expert`).

## Error Handling

- **`_parse_frontmatter`** raises `AssertionError` with the message `"Missing YAML frontmatter delimiters"` if the card text is malformed. In a test context this is a **hard failure** — it means `export_card` produced an unparseable card, which is a bug in the exporter, not a test bug.
- **No try/except anywhere in this file.** All errors propagate as standard pytest failures. There is no swallowing, no retry, no fallback.
- **`yaml.safe_load`** will raise `yaml.YAMLError` if the frontmatter block is not valid YAML. This is another "exporter produced garbage" signal.
- The tests do **not** test error paths of `export_card` (e.g., what happens with a corrupted `Network`). They test the happy path and the empty-path. Error-path testing would live in a separate file or be the responsibility of the exporter's own unit tests.

---

## Topics to Explore

- [file] `reasons/export_card.py` — The implementation under test. Reading it alongside this test file will reveal *how* the frontmatter dict is assembled, how depth is computed (BFS vs. memoized DFS), and how the Markdown template is rendered.
- [file] `reasons/metadata.py` — Defines `SCHEMA_VERSION` and likely the full schema for card frontmatter. Understanding what a "schema bump" means here helps when the `test_schema_version` assertion fails after a library upgrade.
- [file] `tests/test_export_markdown.py` — The sibling exporter test. Comparing the two files shows how the project separates *card* (HuggingFace-publishable) from *markdown* (human-readable) export paths.
- [function] `reasons/export_card.py:export_card` — Focus on the `domain`, `license`, `base_network`, and `source_repos` parameters. Their defaults and the `None`-handling logic are the most likely source of production bugs when users partially configure a card.
- [general] `eem-card-specification` — The HuggingFace EEM card format is an external contract. Exploring the HuggingFace docs or the `SCHEMA_VERSION` migration history clarifies which frontmatter fields are required vs. optional and how versioning works.

## Beliefs

- `export-card-produces-yaml-frontmatter-with-type-eem` — The `type` key in the card frontmatter is always the literal string `"eem"`, identifying the card's taxonomy in the HuggingFace ecosystem.
- `export-card-defaults-license-to-mit` — When the caller does not pass a `license` argument, `export_card` emits `license: mit` in the frontmatter; the test `test_default_license` pins this.
- `export-card-computes-max-depth-as-longest-justification-chain` — The "Max derivation depth" value equals the number of edges in the longest path from a premise to a derived node, not the count of nodes in the chain (0 for premises-only, 1 for premise→derived, 2 for a two-hop chain).
- `export-card-title-uses-project-name-title-case` — The Markdown `# H1` heading is derived from `net.meta["project_name"]` via Python's `.title()` method (e.g., `k8s-ops-expert` → `K8S Ops Expert`).
- `test-file-depends-on-schema-version-constant-not-literal` — The expected `schema_version` value is imported from `reasons.metadata.SCHEMA_VERSION` rather than hardcoded, so a library schema bump requires updating the constant, not the test file.
