---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_export_card.py

# `tests/test_export_card.py` — Explanation

## Purpose

This file is the test suite for `reasons/export_card.py`, which produces a **HuggingFace "EEM" (Expert Enhancement Model) card** — a markdown document with a YAML frontmatter block followed by human-readable metadata, stats tables, and usage instructions. The tests own verification of:

1. **Frontmatter schema** — required keys, defaults, and value types (`schema_version`, `type`, `project_name`, `domain`, `license`, `base_network`, `source_repos`, `generator`).
2. **Belief statistics** — counts of `IN`/`OUT` beliefs, premises vs. derived, and nogoods.
3. **Body sections** — the stats table, retraction rate, "How to Use" section, "Quality" section, and License section.
4. **Derivation depth** — the `Max derivation depth` figure computed from justification chains.
5. **Degenerate inputs** — an empty network must still produce a well-formed card.

In short: it is a **contract test** for the serialized shape of the exported card, split roughly evenly between YAML-structure assertions (via `_parse_frontmatter`) and raw-string assertions on the markdown body.

---

## Key Components

### Helpers

- **`_parse_frontmatter(text)`**
  - Splits the card on `---` with `maxsplit=2`, i.e. `[preamble, yaml_blob, body]`.
  - Asserts `len(parts) >= 3` — a malformed card (missing opening or closing delimiter) fails here with `"Missing YAML frontmatter delimiters"`.
  - Returns `yaml.safe_load(parts[1])` — the frontmatter as a Python `dict`.
  - **Contract**: `export_card()` must always emit a frontmatter block enclosed by `---` lines, otherwise every frontmatter test will hard-fail at this assertion.

- **`_make_network(nodes=None, project_name="test-project", nogoods=None)`**
  - A **network factory/fixture builder**. Handles three concerns:
    1. Sets `net.meta = {"project_name": project_name}` — this is the field later surfaced in the card title and `project_name` frontmatter.
    2. For each `(nid, data)` in `nodes`, creates a `Node(nid, text)` with `truth_value` defaulting to `"IN"` and appends a `Justification` per entry in `data["justifications"]` (default justification `type="SL"`, empty `antecedents`/`outlist`).
    3. Appends `Nogood(**ng)` for each item in `nogoods`.
  - Note: `Node` and `Nogood` are imported **locally inside the helper** — not at module top. This is unusual; it likely avoids collection-time import side effects (or a circular import in the `reasons` package).
  - **Implicit invariant enforced**: a node with no justifications is a **premise**; a node with at least one justification is **derived**. This is how `test_premises_and_derived` distinguishes the two.

### Test classes

- **`TestFrontmatter`** (11 tests) — verifies each frontmatter field and its default:
  - `schema_version == SCHEMA_VERSION` (imported from `reasons.metadata`).
  - `type == "eem"` (constant).
  - `project_name` from `net.meta`.
  - `domain` passed via kwarg, default `[]`.
  - `license` kwarg, default `"mit"`.
  - `base_network` kwarg, default `None`.
  - `source_repos` list passthrough.
  - `generator` contains `"ftl-reasons/"` (soft pinned version string — deliberately a substring check, not an exact match).

- **`TestBeliefCounts`** — verifies the aggregate fields:
  - `beliefs_total`, `beliefs_in`, `beliefs_out` derived from `truth_value`.
  - `premises` vs `derived` split by `justifications` presence.
  - `nogoods` count.

- **`TestEmptyNetwork`** — an empty `Network` still produces a valid card with all counts `0` and a title derived from `project_name`.

- **`TestMaxDepth`** — three cases establishing that `Max derivation depth` equals the **longest chain of antecedent dependency**:
  - 0 for premises only.
  - 1 for `d ← a`.
  - 2 for `c ← b ← a`.
  - Asserted via exact string `"| Max derivation depth | N |"` — so the table cell format is a contract (`| key | value |`).

- **`TestMarkdownBody`** — body-level assertions:
  - Stats table contains `"| Total beliefs | N |"`, `"| Premises (observations) | N |"`.
  - `Retraction rate` is formatted as `N%` (integer percent: 1 OUT / 2 total = `50%`).
  - `## How to Use` section exists and includes the literal command `reasons import-json network.json`.
  - `## Quality` section exists and includes the phrase `"1 beliefs IN, 0 OUT"` (note plural "beliefs" even for 1).
  - `## License` section exists and echoes the license.
  - Title is `# K8S Ops Expert` for `k8s-ops-expert` — implies the implementation does something like `project_name.replace("-", " ").title()`, since Python's `str.title()` uppercases post-digit chars (`"k8s" → "K8S"`).

---

## Patterns

- **Arrange-Act-Assert** per test, with the arrange step compressed into `_make_network`.
- **Test Fixture Factory**: `_make_network` is a lightweight builder with keyword-override defaults, avoiding pytest fixtures for cheap object construction.
- **Frontmatter-split assertion helper**: `_parse_frontmatter` mirrors the classic Jekyll/Hugo convention and gives clear failure messages.
- **Class-based test grouping** by subject (`TestFrontmatter`, `TestBeliefCounts`, `TestMaxDepth`, `TestMarkdownBody`, `TestEmptyNetwork`), enabling focused runs like `pytest tests/test_export_card.py::TestMaxDepth`.
- **Substring assertions on markdown** (rather than full-document snapshots) — deliberately brittle on the *contract* parts (`| Max derivation depth | 2 |`) but lenient about prose. This means small formatting changes in `export_card.py` will need to be reflected here.
- **Dual-surface testing**: YAML frontmatter is validated structurally via `yaml.safe_load`, while body content is validated by string containment.

---

## Dependencies

### Imports (this file depends on)

| Import | Purpose |
|---|---|
| `yaml` (PyYAML) | Parse frontmatter into a dict |
| `reasons.Justification` | Build justification edges in fixtures |
| `reasons.Node`, `reasons.Nogood` | Node and nogood fixtures (imported *inside* `_make_network`) |
| `reasons.metadata.SCHEMA_VERSION` | Contract for `schema_version` frontmatter field |
| `reasons.network.Network` | The data structure being serialized |
| `reasons.export_card.export_card` | The subject under test |

### Imports (depends on this file)

- Only `pytest` (via auto-discovery) — no production code imports tests.

### Coupled production surface

The tests assert on **exact strings** emitted by `export_card`, so the following behaviors in `reasons/export_card.py` are effectively API:
- Frontmatter keys: `schema_version`, `type`, `project_name`, `domain`, `license`, `base_network`, `source_repos`, `generator`, `beliefs_total`, `beliefs_in`, `beliefs_out`, `premises`, `derived`, `nogoods`.
- Body strings: `| Max derivation depth | N |`, `| Total beliefs | N |`, `| Premises (observations) | N |`, `| Retraction rate | N% |`, `## How to Use`, `## Quality`, `## License`, `reasons import-json network.json`, and the `# Title Case` heading.
- `export_card` signature must accept at least the kwargs `domain`, `license`, `base_network`, `source_repos` (all optional).

---

## Flow

Per test, execution is:

```
_make_network(...)                        # build Network with meta/nodes/nogoods
       │
       ▼
export_card(net, **kwargs)                # returns str (markdown + frontmatter)
       │
       ├── _parse_frontmatter(card)       # dict → assert frontmatter fields
       │       └─ text.split("---", 2) → yaml.safe_load(parts[1])
       │
       └── substring asserts on `card`    # body tables, sections, title
```

Notably, the *producer* (`export_card`) is exercised for every test — including pure `TestFrontmatter` cases — so all tests implicitly verify the card is a single self-contained string (no separate file writes).

The derivation-depth logic exercised by `TestMaxDepth` implies the producer performs a recursive or iterative walk over `node.justifications[*].antecedents` to compute the deepest path from a premise, and that premises contribute depth `0`.

---

## Invariants

Rules this file enforces or assumes:

1. **Frontmatter delimiters are always present** — `_parse_frontmatter` requires `card.count("---") >= 2` (via `split("---", 2)` yielding ≥3 parts). A card with only body text will fail every frontmatter test.
2. **`type` is the constant `"eem"`** — this identifies the card flavor for HuggingFace consumers.
3. **`license` defaults to `"mit"`**; `domain` defaults to `[]`; `base_network` defaults to `None`.
4. **`generator` includes a versioned string of form `ftl-reasons/<version>`** — asserted as substring so the version can change freely.
5. **`beliefs_total == beliefs_in + beliefs_out`** is implied but not directly asserted; the mixed test validates all three together (`3 == 2 + 1`).
6. **Premises are nodes with empty `justifications`; derived are nodes with ≥1 justification** — `test_premises_and_derived` encodes this by construction (`premise-a`, `premise-b` have no justifications; `derived-c` has one).
7. **Max derivation depth is 0 for a premise-only network** and increases by 1 per link — enforced by `TestMaxDepth`.
8. **Retraction rate is an integer percentage** — `1 OUT / 2 total` yields `"50%"`, not `"50.0%"` or `"0.5"`.
9. **Empty networks produce a valid, titled card** — no guard in the producer throws on zero nodes/nogoods.
10. **Title-casing replaces hyphens with spaces and title-cases segments** — `k8s` → `K8S`, `my-project` → `My Project`.

---

## Error Handling

- **Assertion-based checks**: `_parse_frontmatter` uses `assert` rather than raising a custom exception. Under `python -O`, asserts are stripped — but test suites never run with `-O`, so this is safe.
- **YAML errors propagate**: A malformed frontmatter block would raise `yaml.YAMLError` from `yaml.safe_load`; the tests do not wrap it.
- **No negative-path coverage**: The suite never tests `export_card` with invalid inputs (e.g., `license=None`, malformed `nodes`, bad `truth_value`), so any validation the producer has is untested here.
- **No exception expectations**: Not one `pytest.raises` appears — this file is entirely positive-path/contract testing.

---

## Topics to Explore

- [file] `reasons/export_card.py` — The producer under test; understand the exact templates behind the asserted string fragments and the derivation-depth algorithm.
- [function] `reasons/export_card.py:export_card` — The public entry point, its `domain`/`license`/`base_network`/`source_repos` keyword args, and default handling.
- [function] `reasons/metadata.py:SCHEMA_VERSION` — The version contract this test pins `frontmatter.schema_version` to.
- [file] `reasons/network.py` — How `Network`, `Node`, `Justification`, and `Nogood` are structured, since fixtures assume `node.justifications[*].antecedents` and `net.meta["project_name"]`.
- [general] `eem-card-format` — What HuggingFace's EEM card format prescribes, to understand why `type: eem`, `base_network`, and `source_repos` exist in the frontmatter.

---

## Beliefs

- `export-card-has-yaml-frontmatter` — `export_card()` output is a single string with a YAML frontmatter block delimited by `---` lines.
- `default-license-is-mit` — Calling `export_card()` without `license=` emits `license: mit` in the frontmatter.
- `card-type-is-eem` — The frontmatter `type` field is always the literal string `"eem"`.
- `premises-defined-by-empty-justifications` — A node is counted as a premise when its `justifications` list is empty; any node with at least one justification is counted as derived.
- `max-derivation-depth-counts-justification-chain-length` — `Max derivation depth` equals the longest chain of antecedent edges in the network (0 for premises-only, 1 for `d ← a`, 2 for `c ← b ← a`).
- `empty-network-produces-valid-card` — `export_card(Network())` succeeds and emits a card with all counts `0` and a title derived from `project_name`.
