---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Explore node/justification structures and the role of `supporting_justification`.

## Node and justification structures

`Node` is defined in `reasons/__init__.py:27`. It holds a node’s identity, text, truth state, and a `justifications` list. The field `supporting_justification` is declared at `reasons/__init__.py:37`:

```python
supporting_justification: int | None = None  # index into justifications list
```

That comment is the key contract: `supporting_justification` is a **positional index into `node.justifications`**, not a database foreign key and not the `justifications.rowid`.

`Justification` is defined in `reasons/__init__.py:7`. The observed schema and tests show it represents one reason for a node, with fields such as:

- `type` — e.g. `"SL"` or `"CP"` in tests
- `antecedents` — node IDs that must be `IN` for the justification to hold
- `outlist` — node IDs that must be `OUT` for the justification to hold
- `label`
- `content_hash`

A node can have multiple justifications. `tests/test_add_justification.py` explicitly verifies this: after calling `net.add_justification("b", ...)`, `len(net.nodes["b"].justifications) == 2`. Those tests also show why multiple justifications matter: after retracting one antecedent, the node can remain `IN` because another justification still supports it.

## Role of `supporting_justification`

`supporting_justification` records **which justification in `node.justifications` is currently the active support for that node**. It is `None` when there is no active justification, such as when a node has no justifications or when truth maintenance clears support.

The network updates this field during propagation/truth maintenance:

- `reasons/network.py:971` assigns `node.supporting_justification = i`, likely when justification index `i` is found valid.
- `reasons/network.py:973` and `reasons/network.py:685` clear it to `None`.

The exact selection algorithm is not visible in the provided snippets, but the call sites and tests imply that when a node becomes `IN`, the network selects a valid justification and records its list index. When the active justification becomes invalid, the network either switches to another valid justification or clears the field.

Consumers rely on this active-reason pointer:

- `reasons/merkle.py:28-31` checks `if not node.justifications or node.supporting_justification is None` and then does `j = node.justifications[node.supporting_justification]`. So Merkle hashing uses the specific active justification, not all justifications.
- `reasons/api.py:893` and `reasons/api.py:1848` serialize `supporting_justification` into API node representations.
- `reasons/api.py:3379`, `reasons/derive.py:126`, `reasons/build_wiki.py:109`, `reasons/build_wiki.py:200`, and `reasons/export_card.py:17` read it to display, derive, or export the supporting reason.
- `reasons/storage.py:22` defines the SQLite column, and `reasons/storage.py:148-149` migrates older databases by adding it. `reasons/storage.py:235-243` saves it, while `reasons/storage.py:357-417` reads it back into `Node.supporting_justification`.

Because it is an index into the in-memory `justifications` list, persistence must keep each node’s justification order consistent between save and load. The `justifications` table has its own autoincrement `rowid`, but `supporting_justification` does not store that rowid.

The provided observations do not include the full bodies around `reasons/network.py:685`, `853`, and `971-973`, nor the full storage load path around `reasons/storage.py:357-417`, so the exact rule for choosing among multiple valid justifications is not visible. The tests show expected behavior, but not the precise selection algorithm.

## Topics to Explore

- [file] `reasons/network.py` — Contains the truth-maintenance code that sets and clears `supporting_justification` and decides when a node becomes `IN` or `OUT`.
- [function] `reasons/network.py:add_justification` — Shows how adding a backup justification can restore an `OUT` node and trigger cascading updates.
- [file] `reasons/merkle.py` — Demonstrates how the active justification is selected for hashing, making `supporting_justification` part of hash determinism.
- [function] `reasons/network.py:trace_assumptions` — Walks justification antecedents backward to premises, which helps explain the justification graph traversal.
- [general] `justification-validity-rules` — The exact semantics of `SL`, `CP`, `antecedents`, and `outlist`, and how a valid justification is chosen as the active one.

## Beliefs

- `supporting-justification-is-index-into-justifications-list` — `Node.supporting_justification` is documented as an integer index into `Node.justifications`, not a foreign key to the `justifications` table (`reasons/__init__.py:37`).
- `node-can-have-multiple-justifications` — `Node.justifications` is a list, and `Network.add_justification` appends to it; tests assert a node can have two justifications and remain supported after one antecedent is retracted.
- `supporting-justification-selects-active-reason-for-merkle` — `reasons/merkle.py:28-31` returns or skips when `supporting_justification` is `None`, otherwise indexes `node.justifications` with it.
- `network-maintains-supporting-justification` — `reasons/network.py:971` assigns an index and `reasons/network.py:973` / `reasons/network.py:685` clear it to `None` during network updates.
- `storage-persists-supporting-justification` — `reasons/storage.py:22` defines the column, `reasons/storage.py:235-243` saves it, and `reasons/storage.py:357-417` reads it back into the node.
