---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: export_card in reasons/export_card.py

# `export_card` — Explanation

## Purpose

`export_card` renders a `reasons` `Network` into a single Markdown string that is meant to be written to `README.md` and published as a **HuggingFace EEM (Evidence & Evaluation Model) card**. The output is a self-contained document containing:

1. A YAML frontmatter block (schema version, project metadata, domain, license, belief statistics, generator).
2. A human-readable Markdown body (title, stats table, usage snippets, file listing, quality summary, limitations, license).

It exists to make a reasoning network *portable and shareable*: a companion `network.json` plus this card form a distributable artifact. The function is pure — it builds a string and returns it; the caller is responsible for writing it to disk.

## Contract

**Preconditions**
- `network` is a fully constructed `Network`. No validation is performed.
- `network.meta`, `network.nodes`, and `network.nogoods` must exist and be `dict`-like / `Mapping`-like / `Sized` respectively.
- Each node must expose `truth_value`, `justifications`, and `supporting_justification`. Each justification must expose `antecedents`.

**Postconditions**
- Returns a `str` (never `None`) whose first line is `---` and which contains at least one terminating `---` line closing the frontmatter.
- When `total > 0`, the string contains a `Retraction rate` row formatted as `NN%`; when `total == 0`, it contains `0%`.
- The string always contains an `# {Title}` heading derived from `project_name`.
- No mutation of `network` occurs.

**Invariants assumed but not enforced**
- `truth_value` values are the string literals `"IN"` / `"OUT"`; anything else is counted as `OUT` (since `out_count = total - in_count`).
- `supporting_justification` is either `None` or an integer index into `node.justifications`.
- `antecedents` entries are valid node IDs in `network.nodes` (unknown IDs simply recurse to "depth 0" — see `_node_depth`).
- The belief graph is acyclic enough for `_node_depth`'s cycle-guard (`memo[nid] = 0` pre-seeded) to be safe.

## Parameters

| Parameter | Type | Default | Meaning / Edge cases |
|---|---|---|---|
| `network` | `Network` | required | The source graph. Must be non-`None`; no check is performed. |
| `domain` | `list[str] \| None` | `None` | YAML `domain:` list. `None` or `[]` → emits `domain: []`. Items are emitted **unquoted** (`  - {d}`), so items containing YAML metacharacters will produce invalid YAML. |
| `license` | `str` | `"mit"` | Emitted **unquoted** as `license: {license}`. A value containing `:` or `#` will corrupt the frontmatter. Also written verbatim into the `## License` body section. |
| `base_network` | `str \| None` | `None` | Parent EEM identifier. `None`/falsy → `base_network: null`. Otherwise double-quoted, but embedded `"` or newlines are not escaped. |
| `source_repos` | `list[str] \| None` | `None` | Emitted as unquoted YAML list items under `source_repos:`. Empty/`None` → `source_repos: []`. |

**Note:** `network.meta.get("project_name", "")` and `.get("created_at", "")` are forwarded to `build_meta`. If a key is *present but `None`*, the `or "eem"` fallback on `meta["project_name"]` still avoids producing an empty title, but no equivalent guard exists for `created_at`.

## Return Value

Always a `str` produced by `"\n".join(lines)`. Structure:

```
---
schema_version: "..."
type: eem
project_name: "..."
domain:           # or []
  - ...
license: mit
base_network: null
source_repos: []
beliefs_total: N
beliefs_in: N
beliefs_out: N
premises: N
derived: N
nogoods: N
generator: ...

# Title

## Stats
...markdown tables...
## How to Use
...bash snippets...
## Files
## Quality
## Limitations
## License
```

Callers must handle: (a) that the returned value is *not* validated YAML if any unquoted field contains special characters; (b) that no trailing newline is appended (the join terminates on the last element, and that element is `""` so effectively it *does* end in the `## License` block plus an empty tail — see `lines` structure).

## Algorithm

1. **Build metadata** — `build_meta(project_name=..., node_count=len(network.nodes), created_at=...)` returns a dict containing (per usage elsewhere in the file) at least `project_name`, `schema_version`, and `generator`.
2. **Fallback project name** — `project_name = meta["project_name"] or "eem"` handles empty/`None`.
3. **Compute stats** over `network.nodes.values()`:
   - `total = len(...)`
   - `in_count` = nodes whose `truth_value == "IN"`
   - `out_count = total - in_count` (so anything not `"IN"` counts as `OUT`)
   - `premises` = nodes with empty/falsy `justifications` (no supporting evidence)
   - `derived = total - premises` (note: *not* filtered by truth value)
   - `nogood_count = len(network.nogoods)`
4. **Compute max derivation depth** — a memoized `_node_depth` is invoked for every node ID; `max(..., default=0)` means an empty graph yields `0`.
   - `_node_depth(nid, nodes, memo)`:
     - returns cached value if present;
     - returns `0` if the node is missing or has no `justifications`;
     - **pre-seeds `memo[nid] = 0`** before recursing (cycle guard — also means the value is wrong if a rejoined node is visited from within its own subtree);
     - if `supporting_justification` is a valid index, restricts traversal to *just that one* justification; otherwise walks *all* justifications;
     - returns `1 + max(depth of protocol antecedents)`.
5. **Retraction percentage** — `out_count / total * 100`, rounded to integer; guarded against `total == 0`.
6. **Assemble YAML frontmatter** — sequential `lines.append(...)` calls; list members are re-emitted without quoting (see Parameter notes).
7. **Assemble Markdown body** — title casing (`-`/`_` → space, `.title()`); fixed stat table; fixed "How to Use" bash snippets; fixed file manifest; quality bullet list derived from stats; static limitations and license.
8. **Return** `"\n".join(lines)`.

## Side Effects

- **None externally.** The only mutation is on the local `memo` dict passed into `_node_depth`. The network is only read.
- No I/O — no file is written. Despite producing a `README.md` payload, the function does not touch the filesystem.
- No logging, no global state.

## Error Handling

No `try/except` and no explicit validation. Possible exceptions:

| Condition | Resulting exception |
|---|---|
| `network` is `None` or lacks `.nodes`/`.meta`/`.nogoods` | `AttributeError` |
| A node lacks `truth_value`, `justifications`, or `supporting_justification` | `AttributeError` |
| A justification lacks `antecedents` | `AttributeError` (raised inside `_node_depth`) |
| `network.nogoods` has no `__len__` | `TypeError` |
| `build_meta` raises on `None`/bad input | Whatever `build_meta` raises |
| `total == 0` | Handled — no `ZeroDivisionError`; `retraction_pct = "0%"`, `max_depth = 0` |
| YAML-invalid characters in unquoted fields | **Not detected** — function happily returns malformed YAML |

Callers that need safety must validate the network before calling (or wrap the call and treat `AttributeError` as a signal of a malformed `Network`).

## Usage Patterns

Typical call from a CLI or batch export:

```python
card = export_card(
    network=n,
    domain=["logic", "education"],
    license="apache-2.0",
    base_network="eem/original-model",
    source_repos=["https://github.com/org/repo"],
)
Path(model_dir, "README.md").write_text(card)
```

**Caller obligations:**
- Write the returned string to `README.md` yourself (the function does not do so).
- Sanitize `license`, `domain` items, and `source_repos` items if they may contain YAML-significant characters (they are emitted raw).
- Do not depend on the exact wording of body sections — they are hard-coded strings, not templates.

## Dependencies

- `build_meta` from `reasons.metadata` — supplies `schema_version` and `generator` that the function reads but never computes itself.
- `Network` from `reasons.network` — type only (no runtime use beyond attribute access).
- `_node_depth` (local) — recursive, memoized depth calculator.
- Standard library: only string formatting; no `json`, `yaml`, or `pathlib`.

## Topics to Explore

- [file] `reasons/metadata.py` — Defines `build_meta`; shows where `schema_version` and `generator` come from and what other keys it returns.
- [function] `reasons/export_card.py:_node_depth` — Memoized, cycle-guarded depth calculation; the `supporting_justification` restriction is the key subtlety.
- [function] `reasons/network.py:Network` — Confirms the shape of `nodes`, `truth_value`, `justifications`, `supporting_justification`, and `nogoods`.
- [file] `tests/test_export_card.py` — Pins expectations for sections, quoting, and empty-network edge cases.
- [general] `eem-card-format` — The HuggingFace EEM card specification to verify the frontmatter and required sections conform.

## Beliefs

- `export-card-returns-markdown-string` — `export_card` always returns a `str` built by joining `lines` with `"\n"`; it never returns `None` and never writes to disk.
- `export-card-handles-empty-network` — An empty `network.nodes` yields `total=0`, `retraction_pct="0%"`, and `max_depth=0` without raising.
- `premises-counted-by-absence-of-justifications` — `premises` is defined as nodes whose `justifications` is falsy, and `derived = total - premises` regardless of truth value, so OUT nodes with justifications count as derived.
- `max-depth-prefers-supporting-justification` — `_node_depth` descends only the `supporting_justification` when that index is valid, falling back to all justifications otherwise.
- `yaml-quoting-is-partial` — Only `project_name` and `base_network` are double-quoted; `license`, `domain` items, and `source_repos` items are emitted unquoted and unescaped, allowing malformed YAML from untrusted input.
