---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: What `supporting_justification` means in Reasons and when it is `None` vs. an out-of-range index.

## What `supporting_justification` means

In Reasons, `supporting_justification` is a **zero-based positional index into `node.justifications`**, not a `Justification` object or a justification ID. The `Node` field is declared as:

```python
supporting_justification: int | None = None  # index into justifications list
```

in `reasons/__init__.py:37`.

Consumers use it directly as a list index. For example, `reasons/merkle.py:31` does:

```python
j = node.justifications[node.supporting_justification]
```

The value is also exported raw through the API (`reasons/api.py:893`, `reasons/api.py:1848`), used by wiki/export/derive code (`reasons/build_wiki.py:109`, `reasons/build_wiki.py:200`, `reasons/derive.py:126`, `reasons/export_card.py:17`), and persisted as a nullable integer in SQLite (`reasons/storage.py:22`, `reasons/storage.py:243`, `reasons/storage.py:417`).

## `None` vs. an out-of-range index

- **`None` is the normal “no selected supporting justification” sentinel.**
  - It is the default value (`reasons/__init__.py:37`).
  - It is stored as SQL `NULL` (`reasons/storage.py:22`) and loaded back as `None` (`reasons/storage.py:417`).
  - Propagation code explicitly clears it to `None` (`reasons/network.py:685`, `reasons/network.py:973`).
  - `merkle.py:28` checks for `node.supporting_justification is None` before indexing, so `None` is a safe, expected state.

- **An out-of-range index is not a valid sentinel; it is stale/corrupt state.**
  - When `supporting_justification` is not `None`, the observed consumers assume it is valid for `node.justifications`.
  - `reasons/merkle.py:31` indexes the list directly with no bounds check. If the index is `>= len(node.justifications)`, it raises `IndexError`. A negative index could also silently select from the end in Python, which is probably not intended.
  - The snippets do not show any code intentionally assigning an out-of-range index. `reasons/network.py:971` assigns an index `i` while selecting a justification, and `reasons/network.py:973` assigns `None` when clearing support. So out-of-range should be treated as a bug/data-integrity issue, e.g. after a justification list mutation that did not update the stored index.

Caveat: the observations include only line-level snippets from `reasons/network.py` around lines 685, 853, and 971–973, not the full function bodies. The exact conditions for selecting index `i` vs. clearing to `None` cannot be fully reconstructed from the provided data. Also, no observed code removes or reorders `node.justifications`, so the precise path that would produce an out-of-range index is not shown.

## Topics to Explore

- [file] `reasons/network.py` — Inspect propagation around lines 685, 853, and 971–973 to see exactly when `supporting_justification` is selected, cleared, or left stale.
- [file] `reasons/merkle.py` — Shows the direct list indexing and the only observed `None`/empty-list guard.
- [file] `reasons/storage.py` — Confirms nullable SQL persistence, migration behavior, and load semantics for the field.
- [general] `justification-list-mutation` — Search for code that removes or reorders `node.justifications`; that is the likely way a stored index becomes out of range.
- [file] `reasons/api.py` — Exports the raw value in API payloads, so external consumers see `None` vs. an integer directly.

## Beliefs

- `supporting-justification-is-list-index` — `Node.supporting_justification` is an integer index into `node.justifications`, not a `Justification` object.
- `none-means-no-selected-justification` — `None` is the default and cleared sentinel meaning no supporting justification is currently selected.
- `merkle-requires-valid-index` — A non-`None` `supporting_justification` must index `node.justifications` or `reasons/merkle.py:31` raises `IndexError`.
- `storage-maps-none-to-sql-null` — `reasons/storage.py` persists `supporting_justification` as a nullable `INTEGER` and loads SQL `NULL` as `None`.
- `network-clears-supporting-justification-to-none` — `reasons/network.py` sets `supporting_justification` to `None` at lines 685 and 973 and assigns an index at line 971.
