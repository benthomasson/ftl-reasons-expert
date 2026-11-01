---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/import_beliefs.py

# `reasons/import_beliefs.py` — Belief & Nogood Importer

## Purpose

This module is the **ingestion bridge** between the human-readable markdown format (used by the `beliefs` CLI) and the internal RMS (Reification-based Management System) network. It owns the responsibility of:

1. Parsing `beliefs.md` (claims with statuses, dependencies, sources) and `nogoods.md` (known-inconsistent sets) from their markdown representations.
2. Converting those parsed structures into `Node` objects with `SL` (Substratum Logic) justifications and `Nogood` records inside a `Network`.

It does **not** create the network, persist it, or reason over it — it only populates an existing `Network` instance. Think of it as a one-way ETL step: markdown in, in-memory graph out.

## Key Components

### `strip_frontmatter(text) → (body, frontmatter_dict)`
- **Contract**: If the text begins with a `---` … `---` block, it splits out the body and returns a flat `key: value` dict. Otherwise returns `(text, {})`.
- **Note**: Deliberately avoids pulling in a YAML library. Only handles simple scalar values (strips surrounding quotes). Nested structures would silently produce garbage — this is a known limitation.

### `parse_repos(text) → dict[str, str]`
- Scans for the `## Repos` section and collects `- name: path` lines until the next `## ` heading.
- Returns an empty dict if the section is absent.

### `parse_beliefs(text) → list[dict]`
- **The core parser.** A line-by-line state machine keyed on the regex `^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)`.
- Each claim dict has a fixed schema: `id`, `status`, `type`, `text`, `source`, `source_url`, `source_hash`, `source_type`, `date`, `depends_on` (list), `unless` (list), `stale_reason`, `superseded_by`, `accepted_pr`.
- The first non-empty, non-metadata line after a claim header becomes `text` (only if `text` is still empty — first-wins).
- Metadata lines are matched by exact prefix strings (`- Source: `, `- Depends on: `, etc.). The `depends_on` and `unless` fields are comma-split into lists.
- Unrecognized `- ` bullets are silently skipped.

### `parse_nogoods(text) → list[dict]`
- Similar line-by-line state machine, keyed on `^### (nogood-\S+?):\s*(.*)`.
- Extracts `discovered`, `resolution`, and `affects` (comma-separated node IDs).

### `import_into_network(network, beliefs_text, nogoods_text=None) → dict`
- **The public entry point.** Orchestrates the full import pipeline:
  1. Strips frontmatter and copies `schema_version`, `project_name`, `created_at` into `network.meta`.
  2. Parses and merges repo mappings into `network.repos`.
  3. Parses claims, topologically sorts them, and inserts them into the network.
  4. Parses and appends nogoods.
  5. Returns a summary dict: `{claims_imported, claims_skipped, claims_retracted, nogoods_imported}`.

## Patterns

| Pattern | Where | Why |
|---|---|---|
| **Line-by-line state machine parser** | `parse_beliefs`, `parse_nogoods` | The markdown format is line-oriented with no nesting; a regex-per-header + prefix-matching loop is simpler and faster than a full markdown AST. |
| **Topological sort (iterative, non-Kahn)** | `import_into_network` | Ensures dependencies are added to the network before their dependents, so SL justifications reference valid node IDs. |
| **Graceful degradation** | Topo-sort fallback, nogood validation | Circular deps or missing references don't crash the import; they're force-added or filtered out. |
| **Idempotent import** | `if claim["id"] in network.nodes: skipped += 1` | Re-running the import on the same network doesn't duplicate nodes. |
| **Builder-style node construction** | `network.add_node(...)` call | All metadata, justifications, and source info are assembled before a single call. |

## Dependencies

**This file imports:**
- `re` — regex for claim/nogood headers
- `pathlib.Path` — imported but **unused** in this file (likely a leftover)
- `Justification`, `Node`, `Nogood` from `.` (the `reasons` package `__init__`)
- `Network` from `.network`

**This file is imported by:**
- `reasons/api.py` — API layer that exposes the import as an endpoint
- `reasons/import_agent.py` — wraps this for agent-driven imports
- `reasons/pg.py` — Postgres-backed dispatch likely calls this before persisting
- `tests/test_import_beliefs.py`, `tests/test_metadata.py`, `tests/test_roundtrip.py`, etc. — test coverage

## Flow

```
import_into_network(network, beliefs_text, nogoods_text)
│
├─ strip_frontmatter(beliefs_text)
│   └─ if frontmatter → copy schema_version / project_name / created_at → network.meta
│
├─ parse_repos(beliefs_text) → network.repos.update(...)
│
├─ parse_beliefs(beliefs_text) → claims: list[dict]
│
├─ Topological sort of claims:
│   ├─ Build claim_by_id lookup
│   ├─ Repeatedly scan `remaining`; a claim is "ready" when all its
│   │   depends_on that exist in the registry have been added
│   ├─ If a full pass makes no progress → force-add the rest (circular deps)
│   └─ Result: `ordered` list
│
├─ For each claim in `ordered`:
│   ├─ If id already in network → skip
│   ├─ Filter depends_on / unless to only IDs present in this batch
│   ├─ If any deps or unless remain → build a single SL Justification
│   ├─ Assemble metadata dict (beliefs_type, source_type, stale_reason, …)
│   ├─ network.add_node(id, text, justifications, source, …)
│   └─ If status is STALE or OUT → network.retract(id)
│
├─ If nogoods_text:
│   ├─ parse_nogoods(nogoods_text)
│   ├─ For each nogood: filter `affects` to valid node IDs
│   ├─ If ≥ 2 valid nodes → append Nogood to network.nogoods
│   └─ Skip duplicates by id
│
└─ return {claims_imported, claims_skipped, claims_retracted, nogoods_imported}
```

## Invariants

1. **Topological ordering is best-effort, not strict.** If claims form a cycle, the sort breaks the cycle by force-adding the remaining claims. Those claims will have SL justifications referencing nodes that may not yet be IN, so the network will compute them as OUT (no valid justification). This is intentional: a circular belief set should not be believed.

2. **STALE and OUT claims are always added-then-retracted.** They must exist as nodes (so other claims' `depends_on` can reference them and be correctly defeated), but their final state must be retracted. The order is: `add_node` → `retract`.

3. **Nogoods require ≥ 2 valid node references.** A nogood with only 0 or 1 resolvable node IDs is silently dropped. This prevents degenerate nogoods that would trivially "defeat" a single node.

4. **Idempotency by ID.** If a node ID or nogood ID already exists in the network, the import skips it. There is no upsert or merge — first writer wins.

5. **`depends_on` and `unless` are filtered to the current batch.** Only IDs present in the parsed claims list (`claim_by_id`) are included in the SL justification. External references (IDs not in this file) are excluded, meaning the justification will have an empty antecedents list and the node will effectively have no justification.

## Error Handling

The design philosophy here is **total non-failure during import**:

- **No exceptions are raised** in the normal code path. Every edge case (missing frontmatter, no Repos section, empty claims, circular deps, missing dependency targets) is handled by producing a best-effort result.
- **`strip_frontmatter`** silently returns the original text if the closing `---` is never found (malformed frontmatter → treated as no frontmatter).
- **Topological sort** detects a stall (no progress in a full pass) and force-appends the remainder. No exception, no infinite loop — bounded by `max_passes = len(remaining) + 1`.
- **Nogood validation** silently skips nogoods with < 2 valid nodes.
- **The summary dict** is the only "error channel" — a developer inspecting `claims_skipped` or finding `nogoods_imported == 0` can infer that something didn't parse as expected.

There is no logging, no `warnings.warn`, and no return of error strings. If the markdown is malformed, you get fewer imports than expected, and that's it. This is a deliberate tradeoff for a batch import operation where partial success is preferable to total failure.

One thing to note for maintainers: the `pathlib.Path` import is unused. Safe to remove.

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class that receives all imported nodes; understanding `add_node`, `retract`, and how SL justifications are evaluated is essential for predicting post-import state.
- [file] `reasons/__init__.py` — Defines `Justification`, `Node`, and `Nogood` dataclasses; their field contracts (especially `Justification.type`, `antecedents`, `outlist`) determine what valid SL justifications look like.
- [file] `tests/test_import_beliefs.py` — The test suite that pins down parsing edge cases (malformed frontmatter, circular deps, duplicate IDs) and the expected summary counts.
- [file] `reasons/import_agent.py` — The caller that feeds agent-generated beliefs through this module; shows how `import_into_network` is used in a live workflow versus the batch CLI path.
- [general] `sl-justification-semantics` — How the RMS network evaluates SL justifications with antecedents and outlists determines whether an imported IN claim actually stays IN, especially when its `unless` targets are also imported as IN.

## Beliefs

- `import-beliefs-is-idempotent-by-id` — Re-running `import_into_network` with the same markdown on an existing network skips all already-present node and nogood IDs; it never overwrites or merges.
- `stale-out-claims-are-added-then-retracted` — Every STALE or OUT claim is first inserted via `add_node` (so dependents can reference it) and then immediately retracted; the node must exist in `network.nodes` even after retraction.
- `topo-sort-breaks-cycles-by-force-adding` — The topological sort in `import_into_network` is bounded and non-throwing; if a cycle is detected (no progress in a full pass), the remaining claims are appended in arbitrary order and will compute as OUT due to unresolvable SL antecedents.
- `nogoods-require-two-or-more-valid-nodes` — A parsed nogood whose `affects` list resolves to fewer than 2 existing node IDs in the network is silently discarded and never appended to `network.nogoods`.
- `depends-on-is-filtered-to-current-batch` — Only `depends_on` and `unless` IDs that appear in the same parsed claims list are included in the generated SL justification; external IDs are excluded, which can leave a node with an empty antecedents list and no effective justification.
