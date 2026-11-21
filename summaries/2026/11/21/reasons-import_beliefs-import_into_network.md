---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: import_into_network in reasons/import_beliefs.py

## Purpose

`import_into_network` imports a parsed `beliefs.md` document, and optionally a `nogoods.md` document, into an existing RMS `Network`. It converts belief claims into RMS nodes, creates SL justifications for `depends_on`/`unless` relationships, retracts claims marked `STALE` or `OUT`, and appends valid nogoods.

It is designed to be a bulk/import operation into a caller-owned `Network`, rather than constructing the network itself.

## Contract

### Preconditions

- `network` is a mutable `Network` instance with at least:
  - `network.meta` as a mutable mapping
  - `network.repos` as a mutable mapping
  - `network.nodes` as a mapping keyed by node ID
  - `network.nogoods` as a mutable list
  - `network.add_node(...)`
  - `network.retract(node_id)`
- `beliefs_text` is a string containing the expected markdown format.
- Claim IDs are assumed to be unique.
- Dependencies in `depends_on` and `unless` are assumed to be claim IDs in the same `beliefs_text` if they are intended to become SL justification antecedents/outlist entries.
- `nogoods_text` is either a string or `None`.

### Postconditions

- Selected frontmatter fields are copied into `network.meta`.
- Parsed repos are merged into `network.repos`.
- New claims are added as nodes unless a node with the same ID already exists.
- Newly imported `STALE`/`OUT` claims are retracted after being added.
- Valid new nogoods are appended to `network.nogoods`.
- A summary dict is returned with import counts.

### Important invariants / assumptions not enforced by types

- Existing node IDs are skipped entirely. They are **not updated** and, importantly, **not retracted even if the imported claim is marked `STALE` or `OUT`**.
- Existing nogood IDs are skipped.
- SL justifications are built only from `depends_on`/`unless` IDs present in the current parsed beliefs batch (`claim_by_id`), not from arbitrary pre-existing nodes in `network.nodes`.
- Topological sorting considers only `depends_on`, not `unless`.
- Nogoods require at least two valid affected node IDs, but those IDs are not deduplicated.
- Duplicate claim IDs are not rejected; `claim_by_id` keeps the last parser occurrence for dependency membership checks.
- The operation is not transactional. If an exception occurs mid-import, `network` may already be partially mutated.

## Parameters

- `network: Network`
  - The target mutable network.
  - Must expose the expected attributes and methods.
  - Missing attributes or incompatible `add_node`/`retract` behavior will raise normal Python exceptions.

- `beliefs_text: str`
  - Markdown text for beliefs.
  - May contain optional YAML-like frontmatter.
  - May contain a `## Repos` section and `### claim-id [IN|OUT|STALE] TYPE` claim headers.
  - Passing `None` is not supported and would fail in `strip_frontmatter` when calling `.startswith()`.

- `nogoods_text: str | None = None`
  - Optional markdown text for nogoods.
  - `None` or an empty string causes nogood import to be skipped.
  - Malformed or under-specified nogoods are parsed leniently and may simply be ignored.

## Return Value

Returns a plain `dict`:

```python
{
    "claims_imported": imported,
    "claims_skipped": skipped,
    "claims_retracted": retracted,
    "nogoods_imported": nogoods_imported,
}
```

- `claims_imported`: number of new claim nodes added via `network.add_node`.
- `claims_skipped`: number of parsed claims whose IDs already existed in `network.nodes`.
- `claims_retracted`: number of newly imported claims that were retracted because their status was `STALE` or `OUT`.
- `nogoods_imported`: number of new nogoods appended with at least two valid affected nodes.

The summary does **not** report:
- skipped duplicate nogoods
- invalid nogoods with fewer than two valid affected nodes
- repo counts
- frontmatter counts
- claims that were ignored by parsing

## Algorithm

1. Strip optional frontmatter from `beliefs_text`.
2. If frontmatter exists, copy only non-empty values for:
   - `schema_version`
   - `project_name`
   - `created_at`
   into `network.meta`.
3. Parse the `## Repos` section and merge results into `network.repos` via `update`.
4. Parse claims from the remaining beliefs markdown.
5. Build `claim_by_id`, a mapping from claim ID to claim dict.
6. Perform an iterative topological sort over claims:
   - A claim is “ready” when all of its `depends_on` IDs that are present in `claim_by_id` have already been added to the `added` set.
   - External or missing dependencies are ignored for ordering.
   - If no progress is made, all remaining claims are appended in their current order, which handles cycles or unresolved dependency ordering gracefully.
7. Iterate through ordered claims:
   - Skip any claim whose ID already exists in `network.nodes`.
   - Build an SL justification if any `depends_on` or `unless` IDs are present in `claim_by_id`.
   - Filter dependency IDs through `claim_by_id`, not through `network.nodes`.
   - Build metadata from type, source type, stale reason, superseded-by, and accepted PR fields.
   - Add the node using `network.add_node(...)`.
   - If status is `STALE` or `OUT`, call `network.retract(claim["id"])`.
8. If `nogoods_text` is present:
   - Parse nogoods.
   - Skip existing nogood IDs.
   - Filter each nogood’s `affects` list to IDs currently present in `network.nodes`.
   - Import only nogoods with at least two valid affected nodes.
   - Append directly to `network.nogoods`.
9. Return the summary counts.

## Side Effects

Mutates the provided network:

- Updates `network.meta`
- Updates `network.repos`
- Adds nodes through `network.add_node`
- Retracts nodes through `network.retract`
- Appends nogoods directly to `network.nogoods`

No file I/O or network access is performed by this function. Input strings are not mutated. The function is not atomic; exceptions can leave the network partially modified.

## Error Handling

There is no explicit exception handling. The function propagates exceptions from:

- Missing network attributes
- `network.add_node(...)`
- `network.retract(...)`
- `Justification(...)`
- `Nogood(...)`
- Type errors such as passing `None` for `beliefs_text`

Parsing helpers are permissive:
- Unrecognized markdown lines are generally ignored.
- Invalid claim headers may simply not be parsed.
- Invalid nogoods may be silently skipped if fewer than two affected nodes remain after filtering.
- Duplicate existing node/nogood IDs are silently skipped rather than reported.

## Usage Patterns

Typical usage:

```python
from reasons.network import Network
from reasons.import_beliefs import import_into_network

network = Network()
summary = import_into_network(
    network,
    beliefs_text=beliefs_markdown,
    nogoods_text=nogoods_markdown,
)
```

Caller expectations:

- Create/configure the `Network` first.
- Re-importing the same file is idempotent for node and nogood IDs, but existing nodes are not refreshed.
- If an existing node should become `OUT`, this function will not retract it because existing nodes are skipped before status handling.
- Pre-existing dependency nodes are not included in new SL justifications unless those dependency IDs are also present in the same imported beliefs text.
- Nogoods only import if at least two referenced affected nodes exist after claim import.

## Dependencies

- `Network` from `reasons.network`
- `Justification` and `Nogood` from the `reasons` package
- Same-module helpers:
  - `strip_frontmatter`
  - `parse_repos`
  - `parse_beliefs`
  - `parse_nogoods`
- Relies heavily on the runtime API and data structures of `Network`, `Justification`, and `Nogood`.

## Topics to Explore

- [file] `reasons/network.py` — Inspect `Network.add_node`, `Network.retract`, and RMS evaluation semantics, especially how nodes with `justifications=None` or SL outlists are handled.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Understand the exact markdown grammar and which claim fields are guaranteed before `import_into_network` runs.
- [function] `reasons/import_beliefs.py:parse_nogoods` — See how nogood IDs, labels, discovery metadata, and `Affects` lists are parsed.
- [file] `tests/test_import_beliefs.py` — Review expected behavior for skips, retractions, dependencies, and import summaries.
- [general] `SL-justification-semantics` — Clarify how RMS uses SL antecedents/outlist nodes and why filtering dependencies to the current import batch matters.

## Beliefs

- `existing-node-ids-are-skipped` — Claims whose IDs already exist in `network.nodes` are not updated or retracted by `import_into_network`.
- `sl-justifications-use-same-batch-dependencies` — SL justifications are built only from `depends_on`/`unless` IDs present in the parsed `beliefs_text`, not from arbitrary pre-existing network nodes.
- `stale-and-out-claims-are-retracted-after-add` — Newly imported claims with status `STALE` or `OUT` are added via `network.add_node` and then passed to `network.retract`.
- `nogoods-require-two-valid-affects` — A parsed nogood is imported only when at least two of its `affects` IDs exist in `network.nodes` after claim import.
- `frontmatter-copies-selected-meta-keys` — Only non-empty `schema_version`, `project_name`, and `created_at` frontmatter values are copied into `network.meta`.
