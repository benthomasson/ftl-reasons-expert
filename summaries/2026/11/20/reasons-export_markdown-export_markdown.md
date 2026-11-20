---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: export_markdown in reasons/export_markdown.py

## `export_markdown` — Function Explanation

### 1. Purpose

`export_markdown` serializes a reasoning network (the in-memory `Network` object used throughout the `reasons` tool) into a single self-contained Markdown string. The output is designed to be a human-readable "beliefs registry" that conforms to a `beliefs.md` schema — i.e., it has structured YAML frontmatter, one Markdown section per claim/node (`### id [STATUS] TYPE`), plus an optional Repos section and an optional Nogoods section.

It exists so the network can be inspected, version-controllable, diffed in Git, and exported to or from other tools that speak the `beliefs.md` dialect. The CLI command `reasons export-markdown` is the obvious caller (see the docstring on the header comment line).

---

### 2. Contract

**Preconditions (not enforced by types):**
- `network` has a `.meta` mapping (e.g., `dict`), a `.nodes` mapping of `id -> node`, and a `.nogoods` iterable (possibly empty).
- Every node exposes `.id`, `.truth_value`, `.text`, `.justifications`, `.metadata` (a mapping), and optionally `.source`, `.source_url`, `.source_hash`, `.date`.
- Each justification exposes `.antecedents` (iterable of node ids) and `.outlist` (iterable of node ids).
- `truth_value` is a string — empirically `"IN"` or `"OUT"`.
- `node.metadata` values that may be read: `stale_reason`, `retract_reason`, `beliefs_type`, `defeats_node`, `defeater_type`, `defeat_reason_type`, `duplicate_of`, `superseded_by`, `accepted_pr`, and `source_type`.
- Nogood objects expose `.id`, `.discovered`, `.resolution`, `.nodes` (iterable of ids).

**Postconditions:**
- Returns a `str` beginning with a `---` … `---` YAML frontmatter block, followed by `# Belief Registry` and a "do not edit" comment line.
- If `repos` is truthy, emits a `## Repos` section before `## Claims`.
- All nodes are emitted under `## Claims`, ordered by IN-first then alphabetical by `id`.
- Nogoods are emitted under `## Nogoods` iff `network.nogoods` is truthy.
- The string is pure: no files are written, no state is mutated.

**Invariants / guarantees:**
- `IN` nodes always appear before non-`IN` nodes. The sort key `(n.truth_value != "IN", n.id)` relies on Python's `bool` being `int` — `False` (0) sorts before `True` (1).
- Every node emits a `### ` heading and its `text`.
- `Depends on`, `Unless`, and `Defeated by` are deduped within their respective buckets.

---

### 3. Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `network` | `Network` | Source graph/registry. | Must expose `.meta`, `.nodes`, `.nogoods`; each node must expose the fields above. If any is missing → `AttributeError`. |
| `repos` | `dict[str, str] \| None` | Optional `repo name -> path` mapping for the `## Repos` section. | `None` or empty dict → section skipped entirely. Keys are sorted alphabetically (`sorted(repos.items())`), so caller-controlled ordering is discarded. |

---

### 4. Return Value

Returns a single `str` (never `None`). The string already ends with a trailing newline because every section finishes with `lines.append("")` before being joined with `"\n"`.

The caller is responsible for:
- Writing the string to a target file or stdout.
- Validating that it is acceptable `beliefs.md`.
- Any post-processing (e.g., newline normalization for the target OS).

---

### 5. Algorithm

1. **Build metadata.** Call `build_meta(project_name=network.meta.get("project_name",""), node_count=len(network.nodes), created_at=network.meta.get("created_at",""))`. This returns a mapping with at least `schema_version`, `project_name`, `updated_at`, `node_count`, `generator`. Note `created_at` is *passed in* but does **not** appear in the emitted frontmatter — presumably `build_meta` uses it another way, or it's a latent omission.

2. **Emit YAML frontmatter** between two `---` lines:
   - `schema_version` (always quoted)
   - `project_name` quoted if non-empty, otherwise emitted as `project_name:` (valid YAML null)
   - `updated_at` (always quoted)
   - `node_count` (unquoted int)
   - `generator` (unquoted)
   
   Then `# Belief Registry` heading and an HTML comment reminding readers not to edit by hand.

3. **Optional Repos section.** If `repos` is truthy, append `## Repos` and one `- name: path` bullet per item.

4. **Claims section header.**

5. **Sort nodes** by `(n.truth_value != "IN", n.id)` — IN first, then everything else; alphabetical by id within groups.

6. **Per node** (`for node in nodes:`):
   - Derive `status` from `node.truth_value`. If `"OUT"` and metadata has `stale_reason` or `retract_reason`, flip to `"STALE"`.
   - Derive `btype`: use `metadata["beliefs_type"]` if set; else `"DERIVED"` if `node.justifications` is non-empty, else `"OBSERVATION"`.
   - Emit heading `### {id} [{status}] {btype}`, then `node.text`.
   - Emit one-line source/date metadata if present: `Source`, `Source URL`, `Source type` (metadata), `Source hash`, `Date`.
   - **Aggregate dependencies across justifications:**
     - `all_deps`: dedupe (order-preserving) of every antecedent from every justification.
     - `all_unless` / `all_defeaters`: for every outlist id `o`, look it up in `network.nodes`. If it exists **and** its `metadata["defeats_node"] == node.id`, it's a defeater; capture `(id, defeater_type, defeat_reason_type)` and skip `unless`. Otherwise, add to `unless`. **Load-bearing quirk:** an id can end up in both `all_unless` and `all_defeaters` if different justifications classify it differently, because the two lists are maintained independently.
   - Emit `- Depends on: a, b`, `- Unless: x, y`, and one `- Defeated by: id (label)` line per defeater (label = `"dtype, rtype"` if `rtype` else `dtype`).
   - Emit `- Stale reason: …` if status is `STALE` or `OUT` **and** a reason exists (from `retract_reason` first, then `stale_reason`).
   - Emit optional `Duplicate of`, `Superseded by`, `Accepted PR`.
   - Blank line after each node.

7. **Nogoods.** If `network.nogoods` is truthy, emit `## Nogoods`, one `### id` per nogood with optional `Discovered`, `Resolution`, and a `- Affects: …` line.

8. `return "\n".join(lines)`.

---

### 6. Side Effects

**None observable to the caller.** The function is pure in the usual sense:
- No I/O (no file writes, no stdout, no network).
- Does not mutate `network`, nodes, justifications, or `repos`.
- Only local Python objects are created.

There is one subtlety: it *relies on read-only access* to `node.metadata`, so if a concurrent thread mutates metadata during the loop, output is undefined — but that's true of any list-like iteration.

---

### 7. Error Handling

No `try/except`. Every failure bubbles up:

| Failure | Trigger |
|---|---|
| `AttributeError` | `network.meta` / `network.nodes` / `network.nogoods` missing; `node.metadata` missing or `None`; missing `node.source`, `node.date`, etc. (only if the attribute truly doesn't exist — the code uses attribute access, not `getattr(..., None)`). |
| `TypeError` | `node.justifications` or `j.antecedents` / `j.outlist` not iterable; sorting mixed-type ids. |
| `KeyError` | Only if `build_meta` returns a dict missing `schema_version`/`updated_at`/`generator` — unlikely given `build_meta`'s contract. |
| `ValueError` | `sorted(repos.items())` with non-comparable values would raise, but keys are strings by type. |

The function does **not** validate that `node.truth_value ∈ {"IN","OUT"}`, that ids are unique, or that antecedent ids exist in `network.nodes`. Antecedents (unlike outlist ids) are emitted *blindly*; they may reference nodes that were never loaded.

---

### 8. Usage Patterns

Typical call site (from the doc comment):

```python
markdown = export_markdown(network, repos={"core": "src/core", "web": "src/web"})
Path("beliefs.md").write_text(markdown)
```

Caller obligations:
- Ensure `network` is fully loaded (all referenced antecedents ideally present, though not required for export).
- Provide `repos` if the target `beliefs.md` needs a Repos section — otherwise the file omits it.
- Treat the result as **text to persist**, not as a partial string — the trailing blank line and `---` framing assume it is written as a whole.
- If `project_name` may contain `"` characters, the caller should sanitize first: the frontmatter does *not* YAML-escape it.

---

### 9. Dependencies

- **Internal:** `reasons.metadata.build_meta` (builds frontmatter metadata dict), `reasons.network.Network` (type only, imported for annotation).
- **Stdlib:** none beyond built-in `sorted`, string formatting, and list/`join`. No `json`, `yaml`, or `pathlib` here — the function emits raw text, so quotes/`:` in values are *not* YAML-quoted/escaped beyond the two explicit `"…"` sites for `schema_version`, `project_name`, and `updated_at`.
- **Downstream assumptions:** the shape of `node.justifications`, `j.antecedents`, `j.outlist`, and the enumerated `metadata` keys are owned by the network and importer/exporter modules — `export_markdown` treats them as duck-typed contracts.

---

### 10. Assumptions Not Enforced by the Type System

- `node.truth_value` is `"IN"` / `"OUT"` — anything else (typo, enum) silently sorts as non-IN and displays verbatim.
- `node.metadata` is always a mapping — a `None` metadata blows up on the first `.get`.
- `node.justifications` may be empty, in which case the node is labeled `OBSERVATION`.
- Outlist `metadata["defeats_node"]` equals the current node id **exactly** — any mismatch means the entry becomes `Unless` rather than `Defeated by`.
- `node.text` is expected to be usable as inline Markdown after a heading; newlines inside `node.text` will break the flat bullet formatting.
- `project_name`, `generator`, and any metadata-derived value used in a `- ...` line are assumed to be free of `\n` and leading characters that would disturb Markdown or YAML.

---

## Topics to Explore

- [file] `tests/test_export_markdown.py` — The canonical tests for this function; they encode the exact formatting expectations (frontmatter, sorting, defeater vs. unless).
- [function] `reasons/metadata.py:build_meta` — Determines `schema_version`, `updated_at`, and `generator` semantics; clarifies why `created_at` is passed but not emitted.
- [file] `reasons/network.py` — Defines `Network.nodes`, `Network.meta`, `Node`, `Justification`, and `Nogood`, which the export duck-types against.
- [file] `tests/test_roundtrip.py` — Implies an import counterpart; useful to understand the export side of the round-trip so the format contract is symmetric.
- [general] `beliefs-md-format` — Understand the external schema this output claims compatibility with, so changes here can be evaluated against a real spec.

---

## Beliefs

- `node-sorting-in-first-then-alphabetical` — Nodes are ordered by `(truth_value != "IN", id)`, producing IN nodes first and alphabetical ids within each group.
- `stale-status-maps-from-out-plus-reason` — A node with `truth_value == "OUT"` and either `stale_reason` or `retract_reason` is relabeled `STALE` in the heading.
- `beliefs-type-defaults-from-justifications` — Missing `beliefs_type` defaults to `DERIVED` when `node.justifications` is non-empty, otherwise `OBSERVATION`.
- `defeater-vs-unless-disambiguated-by-defeats-node` — An outlist id is treated as `Defeated by` only when the corresponding node's `metadata["defeats_node"]` equals the current node's id; otherwise it becomes `Unless`.
- `export-markdown-has-no-side-effects` — The function performs no I/O and mutates no input; it returns a pure `str` for the caller to persist.
- `frontmatter-omits-created-at` — `created_at` is passed into `build_meta` but is not emitted in the YAML frontmatter block.
