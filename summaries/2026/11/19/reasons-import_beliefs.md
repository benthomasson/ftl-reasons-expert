---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/import_beliefs.py

# `reasons/import_beliefs.py`

## Purpose

This module is the **inbound adapter** for the project's markdown persistence format. It reads `beliefs.md` and `nogoods.md` (the human-editable, git-friendly representation of an RMS — a "Reason Maintenance System" network) and hydrates an in-memory `Network` object with nodes, justifications, and nogoods. It is the counterpart of `reasons/export_markdown.py`, which serializes the network back to the same format.

Its responsibility is narrow: **parse → order → construct `Node`/`Justification`/`Nogood` objects → mutate a given `Network`**. It does not own storage, does not open files (that's the caller's job — note `Path` is imported but unused), and does not emit anything beyond a summary dict.

It is the canonical loader used by `reasons/api.py`, `reasons/import_agent.py`, and `reasons/pg.py`, and is mirrored verbatim (for the time being) in `reasons_lib/import_beliefs.py`.

## Key Components

### `strip_frontmatter(text) -> (body, fm_dict)`
Detects a leading `---` … `---` block and returns the body plus a flat `dict`. This is a deliberately minimal parser: single `key: value` per line, strips surrounding `"`/`'` quotes. No YAML library, no nesting.

- If the opening `---` has no matching closing `---`, the whole text is returned unchanged with `{}`.
- Blank/non-colon lines inside the frontmatter are silently ignored.

### `parse_repos(text) -> dict[str, str]`
Scans for a `## Repos` heading, then collects `- name: path` bullets until the next `## ` heading. Splits on the **first** `:` only, so paths containing colons survive.

### `parse_beliefs(text) -> list[dict]`
Line-oriented parser keyed off the header regex:

```
^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)
```

Each claim is a `dict` with the schema:
`{id, status, type, text, source, source_url, source_type, source_hash, date, depends_on, unless, stale_reason, superseded_by, accepted_pr}`.

Metadata field recognition is **inconsistent by design/case**:
- `- Source URL:`, `- Source hash:`, `- Accepted PR:` are **case-sensitive** (`startswith`).
- `- Stale reason:` / `- Superseded by:` use `.lower().startswith(...)` so any capitalization works.

Text body: only the **first** non-empty, non-`- `-prefixed line after the header is captured (see line `elif line.strip() and not current["text"]`). Multi-line claim text is silently truncated to a single line.

### `parse_nogoods(text) -> list[dict]`
Header regex `^### (nogood-\S+?):\s*(.*)` requires the `nogood-` prefix. Captures `id`, `label`, `discovered`, `resolution`, `affects`. **Note: `label` is parsed but never used** downstream — `Nogood(...)` is constructed without it.

### `import_into_network(network, beliefs_text, nogoods_text=None) -> dict`
The orchestration entry point. Mutates the passed `Network` and returns:

```python
{
  "claims_imported": int,
  "claims_skipped": int,
  "claims_retracted": int,
  "nogoods_imported": int,
}
```

The return is a plain counters dict — no exceptions on partial failure, no error list.

## Patterns

- **Parse-then-apply separation.** Every parser is pure (`text → list/dict`); the only side-effecting function is `import_into_network`. This makes the parsers trivially unit-testable.
- **Idempotent import.** Both nodes and nogoods are checked against existing IDs (`claim["id"] in network.nodes`, `existing_nogood_ids`) before insertion. Re-running is a no-op for already-present items (except retraction, which is idempotent in most RMS implementations).
- **Topological sort with a cycle fallback.** Manual O(n²)-er iterative Kahn-style sort using `claim_by_id`, `added`, and repeated passes with `max_passes = len(remaining) + 1`. If a pass makes no progress, all remaining claims are appended anyway ("will be computed as OUT if deps are missing" per the inline comment).
- **Status → behavior mapping** (the module's docstring codifies this contract):
  - `IN` + no deps → premise (no justifications)
  - `IN` + deps → SL justification
  - `STALE` → add then `network.retract(id)`
  - `OUT` → add then `network.retract(id)`
- **Metadata passthrough.** Only a whitelisted set of optional fields are serialized into `metadata`: `beliefs_type`, `source_type`, `stale_reason`, `superseded_by`, `accepted_pr`. Empty values are dropped to avoid polluting metadata with `""`.
- **SL (Standard/Defeasible Logic) justification shape** — `Justification(type="SL", antecedents=[...], outlist=[...], label=...)` where `outlist` is derived from `unless`.

## Dependencies

**Imports (upstream):**
- `re` — the only external lib used (two regexes).
- `Path` — **unused** (imported for symmetry / historical reasons).
- `Node` — **unused** directly (constructed inside `Network.add_node`).
- `Justification`, `Nogood` — constructed here.
- `Network` — type hint and duck-typed receiver; the module relies on `network.meta`, `network.repos`, `network.nodes`, `network.nogoods`, `network.add_node(...)`, `network.retract(id)`.

**Consumers (downstream):**
- `reasons/api.py`, `reasons/import_agent.py`, `reasons/pg.py` — the production entry points.
- `reasons_lib/import_beliefs.py` — packaged mirror.
- Tests: `test_import_beliefs.py`, `test_metadata.py`, `test_metadata_extended.py`, `test_nogood_id.py`, `test_roundtrip.py`, `test_source_type.py` — the round-trip test (`test_roundtrip.py`) is the strongest contract guard: `export_markdown → import_beliefs` must be lossless for the fields this module knows about.

## Flow

```
import_into_network(network, beliefs_text, nogoods_text)
├─ strip_frontmatter(beliefs_text) → (body, fm)
│  └─ if fm has schema_version / project_name / created_at:
│       network.meta[key] = fm[key]
├─ parse_repos(body) → network.repos.update(...)
├─ parse_beliefs(body) → claims[]
├─ claim_by_id = {c["id"]: c for c in claims}   # last-wins on duplicate IDs
├─ iterative topological sort of claims → ordered[]
└─ for claim in ordered:
     ├─ skip if claim["id"] in network.nodes
     ├─ deps_in_network = [d for d in depends_on if d in claim_by_id]
     ├─ unless_in_network = [u for u in unless if u in claim_by_id]
     ├─ if deps or unless: justifications = [SL(...)]
     ├─ metadata = whitelist({beliefs_type, source_type, stale_reason, superseded_by, accepted_pr})
     ├─ network.add_node(id, text, justifications, source, source_url,
     │                    source_hash, date, metadata)
     └─ if status in {"STALE","OUT"}: network.retract(id)
└─ if nogoods_text:
     ├─ existing = {ng.id for ng in network.nogoods}
     ├─ for ng in parse_nogoods(nogoods_text):
     │    ├─ skip if ng["id"] in existing
     │    ├─ valid_nodes = [a for a in ng["affects"] if a in network.nodes]
     │    ├─ if len(valid_nodes) >= 2: append Nogood(id, nodes=valid_nodes,
     │    │                                      discovered, resolution)
     │    └─ else: silently drop
     └─ nogoods_imported = count
```

## Invariants

1. **Topological order.** A dependency that appears in the same file is always added before the claim that references it, so `network.add_node` never needs to resolve forward references from the file. This is required for the SL `antecedents` to point at real nodes at insert time.
2. **In-file-only justifications.** `deps_in_network` / `unless_in_network` filter by `claim_by_id` membership, **not** by `network.nodes` membership. Consequence: a claim whose `depends_on` points to a pre-existing network node that is *not* re-listed in this file will receive `justifications=None` and be imported as a **premise**. Deps are *not* aggregated across file boundaries.
3. **Nogoods need ≥ 2 live nodes.** `if len(valid_nodes) >= 2` is a hard gate; single-node or all-missing nogoods are dropped silently.
4. **Status retraction.** `STALE` and `OUT` both produce an added node that is immediately retracted. The distinction between them survives only as `metadata["stale_reason"]` / `metadata["superseded_by"]` (which are populated only when the source file carries those fields).
5. **Duplicate claim IDs collapse.** `claim_by_id` is a last-wins dict; `parse_beliefs` may append multiple entries with the same ID, but only the last will be ordered and pushed through (the others silently lose).
6. **Metadata is value-filtered.** Empty strings never enter `metadata` — enabling round-trip tests that assert absence rather than presence of empty keys.
7. **Frontmatter keys are whitelisted.** Only `schema_version`, `project_name`, `created_at` are lifted into `network.meta`; arbitrary frontmatter keys are dropped.

## Error Handling

There is **no explicit error handling** — the module is deliberately permissive:

- **Missing/circular deps:** the sort loop's `if len(next_remaining) == len(remaining)` fallback appends the remainder unconditionally; the inline comment claims the RMS will evaluate such nodes as `OUT`.
- **Unparseable frontmatter** (no closing `---`): returns the text untouched, no warning.
- **Unknown status tokens** (`### x [FOO]`): the regex `(IN|OUT|STALE)` fails to match, so that header is treated as ordinary body text and the claim is **not** created — a silent drop.
- **Unknown nogood IDs / invalid affects:** imported nogoods missing required nodes are dropped without surfacing an error.
- **Duplicate insertion:** existing node/nogood IDs are simply skipped and counted.
- **No I/O.** The module never touches the filesystem; callers that pass a bad path fail in `Path.read_text()` upstream, not here.

The **summary dict is the only feedback channel** — a caller that needs strict-mode validation for malformed input would have to pre-parse and inspect `claims_skipped` / `nogoods_imported` versus its own expectations.

## Topics to Explore

- [file] `reasons/network.py` — Defines `Network.add_node`, `retract`, `repos`, `meta`, `nodes`, `nogoods`; every contract this module relies on is anchored here.
- [file] `reasons/export_markdown.py` — The inverse serializer; together with `import_beliefs` it defines the on-disk format, and `test_roundtrip.py` is the guardrail.
- [function] `reasons/network.py:Network.add_node` — Resolves how `justifications=None` becomes a premise (vs. an OUT node) and whether antecedents are required to exist at call time or resolved lazily.
- [file] `tests/test_roundtrip.py` — Enumerates exactly which fields the import/export pair must preserve; it's the de facto spec for this module.
- [general] `rms-justification-semantics` — How `SL` justifications with `outlist` implement `unless` / defeasibility in this RMS; this determines what "computed as OUT" actually means for the cycle fallback.

## Beliefs

- `import-is-idempotent` — `import_into_network` skips any claim whose `id` already exists in `network.nodes` and any nogood whose `id` already exists in `network.nogoods`, so repeated imports do not duplicate entries.
- `topological-order-prevents-forward-references` — Claims are topologically sorted by `depends_on` before insertion, guaranteeing in-file dependencies are added to the network before the claims that reference them.
- `sl-justification-references-only-in-file-deps` — The SL justification's `antecedents` are filtered by `claim_by_id` membership, so a claim depending only on pre-existing network nodes gets `justifications=None` and is imported as a premise.
- `stale-and-out-claims-are-retracted-after-add` — Both `STALE` and `OUT` statuses route through `network.retract(id)` immediately after `add_node`, and both increment `claims_retracted`.
- `nogoods-require-at-least-two-live-nodes` — A nogood is only appended if `len([a for a in affects if a in network.nodes]) >= 2`; otherwise it is dropped silently.
- `frontmatter-whitelist-maps-three-keys` — Only `schema_version`, `project_name`, and `created_at` from YAML frontmatter are copied into `network.meta`; other keys are ignored.
