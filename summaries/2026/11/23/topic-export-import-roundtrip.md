---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: `api.export_network` / `api.import_json` schema, to see how metadata like `scope_of`, `for_defeater`, `defeats_node` survives serialization

# How `api.export_network` / `api.import_json` serialize metadata

## Short answer

`scope_of`, `for_defeater`, and `defeats_node` are **not special top-level serialization fields**. They are ordinary keys inside each node’s generic `metadata` dictionary.

The survival path is:

1. `defeat_with_scope` / related API code creates nodes with metadata such as:
   - `scope_of`: set at `reasons/api.py:1495`
   - `for_defeater`: set at `reasons/api.py:1496`
   - `defeats_node`: set at `reasons/api.py:1380`, `1507`, and `1622`
2. That metadata is stored on `Node.metadata`.
3. `export_network` (`reasons/api.py:1814`) serializes each node, including its `metadata` dict, into JSON-compatible data.
4. `import_json` (`reasons/api.py:2104`) reconstructs nodes and must pass that metadata back into `Node(metadata=...)` / `Network.add_node(metadata=...)`.
5. Because `metadata` is an opaque dictionary, arbitrary keys like `scope_of`, `for_defeater`, and `defeats_node` survive as long as export copies the dict and import restores it.

The important caveat: the observations do **not** include the full bodies of `export_network` or `import_json`, so the exact top-level JSON schema and exact import implementation are inferred from tests plus `Network.add_node`.

---

## Inferred / observed export schema

Tests directly confirm that `export_network` returns a dictionary with a `nodes` mapping keyed by node ID:

```python
export = api.export_network(db_path=str(db))
export["nodes"][result["defeater_id"]]
```

See `tests/test_defeat_with_scope.py::test_defeat_with_scope_in_export`.

The node payload appears to include fields from `Network.add_node` in `reasons/network.py`, plus `truth_value` and `justifications`.

A representative exported node looks like:

```json
{
  "nodes": {
    "defeater-1": {
      "text": "Defeater text",
      "truth_value": "IN",
      "justifications": [
        {
          "type": "SL",
          "antecedents": ["scope-p1-defeater-1"],
          "outlist": [],
          "label": ""
        }
      ],
      "source": "",
      "source_url": "",
      "source_hash": "",
      "date": "",
      "metadata": {
        "defeats_node": "d1",
        "defeats_justification": 0
      },
      "created_at": "...",
      "updated_at": "...",
      "reviewed_at": "",
      "verified_at": "",
      "retracted_at": ""
    }
  },
  "nogoods": [
    ["a", "b"]
  ]
}
```

The exact presence of all timestamp/source fields in the exported JSON is inferred from `Network.add_node` (`reasons/network.py`) and tests such as `test_round_trip_premises`, `test_round_trip_justifications`, and `test_round_trip_metadata`. The observations do not show the literal `export_network` body, so other top-level keys may exist.

`import_json` returns counters such as:

```json
{
  "nodes_imported": 2,
  "nogoods_imported": 1
}
```

Confirmed by `tests/test_import_json.py::test_round_trip_premises` and `test_round_trip_nogoods`.

---

## Where the metadata is created

In `defeat_with_scope`-style code, the scope belief node gets:

```python
"scope_of": ant_id,
"for_defeater": defeater_id,
```

at `reasons/api.py:1495-1496`.

The defeater node gets:

```python
"defeats_node": node_id,
```

at `reasons/api.py:1507`, and the tests also assert:

```python
defeater["metadata"]["defeats_node"] == "d1"
defeater["metadata"]["defeats_justification"] == 0
```

in `tests/test_defeat_with_scope.py::test_defeater_is_derived`.

Similarly, `tests/test_defeat_with_scope.py::test_scope_beliefs_are_premises` verifies:

```python
scope["metadata"]["scope_of"] == "p1"
scope["metadata"]["for_defeater"] == result["defeater_id"]
```

So these keys are created as normal `Node.metadata` entries. They are not modeled as separate schema columns or dedicated JSON fields.

Other API code also creates or reads `defeats_node` metadata:

- `reasons/api.py:1380` sets `"defeats_node": node_id`
- `reasons/api.py:1622` sets `"defeats_node": nid`
- `reasons/api.py:1661-1699` scans metadata via `meta.get("defeats_node")`
- `reasons/build_wiki.py:90` and `reasons/export_markdown.py:86` read `metadata.get("defeats_node")`

This confirms `defeats_node` is part of the generic metadata contract, not a serialization-specific field.

---

## How the metadata survives export/import

### Export

`export_network` at `reasons/api.py:1814` loads the network and serializes nodes. For metadata to survive, the serializer must copy each node’s `metadata` dict into the exported node payload.

The test `tests/test_defeat_with_scope.py::test_defeat_with_scope_in_export` demonstrates that the defeat graph survives JSON export/import at least structurally:

- the generated scope belief ID is present in `export["nodes"]`
- the generated defeater ID is present in `export["nodes"]`
- the defeater’s `justifications[0]["antecedents"]` equals `[scope_id]`
- after writing the export to JSON and importing it into a fresh DB, the original target `d1` is still `OUT`

That last assertion matters because the defeat relationship depends on the scope belief/defeater graph and `defeats_node` metadata.

### Import

`import_json` at `reasons/api.py:2104` reads the JSON file and reconstructs the network. The observed tests require it to:

- raise `FileNotFoundError` for a missing file: `test_missing_file_raises`
- skip already-existing node IDs: `test_skip_duplicates`
- restore node metadata: `test_round_trip_metadata`
- restore justifications, including outlists: `test_round_trip_outlist`
- restore retracted/OUT state: `test_round_trip_retracted_state`
- restore nogoods: `test_round_trip_nogoods`

For metadata specifically, `tests/test_import_json.py::test_round_trip_metadata` does:

```python
api.challenge("a", "reason", db_path=db1)
api.defend("a", "challenge-a", "defense", db_path=db1)

data = api.export_network(db_path=db1)
...
api.import_json(json_file, db_path=db2)

node_a = api.show_node("a", db_path=db2)
assert "challenges" in node_a["metadata"]
```

That proves generic node metadata survives the export/import cycle. Since `scope_of`, `for_defeater`, and `defeats_node` are also stored in the same `metadata` dict, they follow the same path.

However, there is no direct test in the observations that asserts:

```python
imported_scope["metadata"]["scope_of"] == "p1"
imported_scope["metadata"]["for_defeater"] == defeater_id
imported_defeater["metadata"]["defeats_node"] == "d1"
```

after `import_json`. The existing defeat export test checks graph structure and final truth value, not those exact metadata keys after import. So the strongest direct evidence for exact-key survival is:

- the keys are created in `Node.metadata`
- generic metadata round-trip is tested
- the export test shows the relevant scope/defeater graph survives

But an explicit exact-key round-trip test is missing from the observations.

---

## Important implementation detail: double encoding of the defeat relationship

The defeat relationship is represented in two places:

1. **Metadata pointers**
   - scope belief:
     - `metadata["scope_of"]`
     - `metadata["for_defeater"]`
   - defeater:
     - `metadata["defeats_node"]`
     - `metadata["defeats_justification"]`

2. **Justification graph edges**
   - the defeater’s SL justification has `antecedents` equal to the generated scope belief IDs:
   
   ```python
   assert j["type"] == "SL"
   assert j["antecedents"] == result["scope_belief_ids"]
   ```

   See `tests/test_defeat_with_scope.py::test_defeater_is_derived`.

This means serialization must preserve both:

- the generic `metadata` dict
- the normal `justifications` list, including `antecedents`

`test_defeat_with_scope_in_export` directly verifies the justification edge:

```python
assert defeater_data["justifications"][0]["antecedents"] == [scope_id]
```

If `import_json` restored justifications but dropped metadata, the graph would exist but API functions that scan for `meta.get("defeats_node")` at `reasons/api.py:1661-1699` would no longer find the defeater relationship correctly. Conversely, if import restored metadata but dropped justifications, the defeater would lose its support edge to the scope belief.

---

## What is missing from the observations

To be 100% precise about the schema, we would need:

- the full body of `reasons/api.py:1814` `export_network`
- the full body of `reasons/api.py:2104` `import_json`
- the serialization/deserialization code in `reasons/storage.py`

Those would confirm:

- exact top-level JSON keys
- whether `repos`, `log`, and `meta` are exported
- whether import uses `Network.add_node(metadata=...)` or constructs `Node` directly
- exactly how truth values and retracted state are restored
- whether unknown metadata keys are copied verbatim or filtered

Based on the observations, the answer is still clear: these metadata keys are ordinary `Node.metadata` entries and should survive as long as `export_network` includes `metadata` and `import_json` restores it.

---

## Topics to Explore

- [function] `reasons/api.py:export_network` — Read the full body to confirm the exact top-level JSON schema and whether `metadata`, `repos`, `log`, and `meta` are copied verbatim.
- [function] `reasons/api.py:import_json` — Read the full body to verify how nodes, justifications, `truth_value`, retracted state, and `metadata` are restored.
- [file] `reasons/storage.py` — Understand how `Node.metadata` is persisted to and loaded from SQLite before `export_network` ever serializes it.
- [function] `reasons/api.py:defeat_with_scope` — See the exact creation of `scope_of`, `for_defeater`, `defeats_node`, and `defeats_justification`.
- [general] `metadata-roundtrip-tests` — Add or inspect tests that assert `scope_of`, `for_defeater`, and `defeats_node` explicitly after `import_json`, since current tests mainly prove generic metadata and defeat graph/outcome preservation.

---

## Beliefs

- `export-network-returns-nodes-mapping` — `api.export_network` returns a dict whose `nodes` key maps node IDs to serialized node payloads, as used by `export["nodes"][defeater_id]`.
- `defeat-metadata-lives-in-node-metadata` — `scope_of`, `for_defeater`, and `defeats_node` are stored inside each node’s `metadata` dictionary rather than as dedicated top-level JSON fields.
- `defeater-justifications-reference-scope-beliefs` — The defeater node’s SL justification `antecedents` list contains the generated scope belief IDs.
- `import-json-restores-generic-metadata` — `tests/test_import_json.py::test_round_trip_metadata` verifies that at least one arbitrary metadata key survives export/import.
- `exact-defeat-metadata-roundtrip-not-directly-tested` — The observed tests do not explicitly assert `scope_of`, `for_defeater`, or `defeats_node` values after `api.import_json`.
