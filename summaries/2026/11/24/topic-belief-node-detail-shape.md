---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The formatter relies heavily on fields like `truth_value`, `justifications`, `supporting_justification`, `dependents`, and `metadata`.

## How the formatter leans on core node fields

The formatter code splits across several modules. The main output surfaces are:

- `reasons/export_markdown.py:export_markdown` — human-readable `beliefs.md`
- `reasons/export_card.py:export_card` — HuggingFace-style dataset card
- `reasons/api.py:_format_markdown`, `_format_json`, `_format_minimal`, `_format_compact`, `_format_names_only` (lines 3282, 3313, 3338, 3350, 3362) — query-result formatters
- `reasons/build_wiki.py:_format_node` (line 53) and `_format_beliefs_for_prompt` (line 188)

Across all of them, five fields do most of the work.

### `truth_value` — the primary status signal

- **Defined** at `reasons/__init__.py:35` as `truth_value: str = "IN"  # IN or OUT`.
- **`export_markdown.export_markdown`** uses it three ways: sorting nodes (`sorted(..., key=lambda n: (n.truth_value != "IN", n.id))`), mapping to a display status (`status = node.truth_value`, upgraded to `"STALE"` when OUT + metadata has a reason), and embedding it in the heading `### {node.id} [{status}] {btype}`. It also gates the retraction-reason line: `if status in ("STALE", "OUT") and retract_reason:`.
- **`export_card.export_card`** counts IN vs OUT directly: `in_count = sum(1 for n in network.nodes.values() if n.truth_value == "IN")` and then emits those stats into the frontmatter and Stats table.
- **`reasons/api.py`** uses it pervasively for status checks and serialization — see lines 262, 267, 398, 488–491, 532, 545, 563–565, 599, 612, 630–632, 667, 687, 707–715, 766–769, 823, 840, 848. The delta-tracking pattern (`before = {nid: n.truth_value ...}`) and `went_in` / `went_out` classification at 488–491 and 766–769 show that formatters reporting "changed" rely on comparing this field before/after.

### `justifications` — the structural source of dependencies

- **`export_markdown.export_markdown`** iterates the list to flatten dependencies: `for j in node.justifications: for a in j.antecedents: ...` and separately over `j.outlist` to split defeaters vs. exceptions using `o_node.metadata.get("defeats_node") == node.id`.
- **`export_card.export_card`** uses an empty check to classify nodes: `premises = sum(1 for n in ... if not n.justifications); derived = total - premises`, which drives both stats and the "Quality" bullet about SL justifications.
- **`reasons/merkle.py:28–31`** guards on `node.justifications` before indexing into it with `supporting_justification`.
- **`reasons/network.py`** mutates it (line 685 hides the referenced field, 971 sets `supporting_justification` after evaluation), and **`build_wiki.py`** destructures it via `node_detail.get("supporting_justification")` at lines 109 and 200 when writing wiki text.

### `supporting_justification` — which justification is authoritative

- **Defined** at `reasons/__init__.py:37` as `int | None = None  # index into justifications list`.
- **`export_card._node_depth`** (lines 8–21) uses it as a selector: if it's set and in range, depth recursion follows only `justifications[si]`; otherwise it walks every justification. This directly changes `max_depth` in the published card.
- **`reasons/merkle.py:31`** uses it to pick the single justification hashed into the Merkle proof.
- **`reasons/build_wiki.py:109`** and **`:200`** read it (`si = node_detail.get("supporting_justification")`) to render "because" chains in prose/wiki output.
- **`reasons/network.py:685`** clears it to `None` (likely on retract/invalidate), and **`:971`/`:973`** set it during evaluation — demonstrating it is a live field maintained by the engine, not just a storage artifact.
- **Persisted** in `reasons/storage.py:22` as a nullable INTEGER, migrated in `:148–149`, written in `:243`, read in `:362` and **reasons/api.py:893, 1848** for API responses, and consumed in **api.py:3379** inside one of the `_format_*` functions.

### `metadata` — where the formatter finds labels and status hints

- **Node-level** `node.metadata.get(...)` is the primary cross-cutting lookup in `export_markdown.export_markdown`. Specific keys read: `stale_reason`, `retract_reason`, `beliefs_type`, `source_type`, `defeats_node`, `defeater_type`, `defeat_reason_type`, `duplicate_of`, `superseded_by`, `accepted_pr`.
- **Network-level** `network.meta.get(...)` is fed into `build_meta` by both exporters — `export_markdown.py` uses `project_name`, `created_at`, `node_count`; `export_card.py` additionally derives totals, retraction percentage, and premises vs. derived counts from the same source.
- **`reasons/metadata.py:build_meta`** centralizes the keys (`schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, `generator`) so the markdown frontmatter and the card frontmatter stay consistent.
- Node metadata also gates which block is emitted — e.g., `if node.metadata.get("source_type"):` and `if node.metadata.get("duplicate_of"):` in `export_markdown.py`.

### `dependents` — insufficient evidence

There is **no direct observation of a `dependents` field** in the supplied data. The closest signal is `export_markdown.export_markdown`'s "Collect all antecedents across justifications as depends_on" block, which builds the inverse relation *inline* by walking `j.antecedents`. If `dependents` exists on the model, its consumers aren't visible here; that gap should be filled by grepping the class definition and `Network` internals.

### What the data does *not* show

- The `Belief` class definition (`find_belief_class` returned `count: 0`), so the exact type of `justifications` and `metadata` isn't confirmed — only usage.
- Bodies of `_format_json`, `_format_minimal`, `_format_compact`, `_format_markdown`, `_format_names_only` (api.py 3282–3362) are not quoted, so their per-field usage is inferred from surrounding lines (e.g. 848 serializes `truth_value`, 3379 reads `supporting_justification`).
- No observation of `dependents` field definition or usage.
- No `_format_*` output samples, so schema of the JSON/compact outputs is unverified.

## Topics to Explore

- [file] `reasons/__init__.py` — Node/Belief data model: confirms types of `truth_value`, `justifications`, `supporting_justification`, `metadata`, and whether `dependents` exists
- [function] `reasons/api.py:_format_json` — The canonical JSON serialization used by query results; shows which fields are emitted versus stripped
- [function] `reasons/network.py:Network` — Engine logic that populates `justifications` and maintains `supporting_justification`; explains the lifecycle the formatter depends on
- [file] `reasons/storage.py` — Persistence layer showing which node fields survive a round-trip through SQLite/Postgres (lines 22, 148–149, 235–243, 357–417)
- [general] `dependents-field-usage` — Locate where `dependents` is defined and consumed; the current observations omit it entirely

## Beliefs

- `truth-value-drives-sort-and-status` — `export_markdown.py` sorts nodes by `(truth_value != "IN", id)` and derives the display status from `truth_value` before writing the heading.
- `supporting-justification-selects-justification` — `export_card._node_depth` and `merkle.py` use `supporting_justification` as an index to select one entry from `justifications`.
- `metadata-is-the-formatter-label-source` — `export_markdown.export_markdown` reads at least nine distinct `node.metadata` keys (`stale_reason`, `retract_reason`, `beliefs_type`, `source_type`, `defeats_node`, `defeater_type`, `defeat_reason_type`, `duplicate_of`, `superseded_by`, `accepted_pr`).
- `justifications-distinguish-premises-from-derived` — Both `export_markdown` and `export_card` classify nodes as premises vs. derived based solely on whether `justifications` is empty.
- `dependents-usage-unverified` — The supplied observations do not contain a definition or usage of a `dependents` field, so any claim about it is unsupported.
