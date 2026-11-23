---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: how `source` strings are generated at import time (agent vs. expert vs. bare path)

## How `source` strings are generated at import time

### Short answer

In the observed code, `source` is **not** synthesized with `agent:` or `expert:` prefixes.

- For normal belief imports, `source` comes directly from the input file’s `- Source: ...` line.
- For agent imports, the imported belief’s own `source` field is preserved from markdown or JSON.
- Agent identity is stored separately: in node IDs like `agent:active`, in metadata such as `{"agent": agent_name}`, and sometimes in labels like `imported from agent: {agent_name}`.
- The only agent-related `source` value generated at import time is for the synthetic `agent:active` / `agent:inactive` nodes: they use the `source_path` argument passed to `_ensure_agent_nodes`, which is a bare path, not an `agent:` or `expert:` prefix.
- There is no observed special handling for `expert:` source strings. “Expert” appears as part of example agent names such as `aap-expert` / `rhel-expert`, not as a source prefix convention.

### Normal belief import: source is parsed literally

`parse_beliefs()` in `reasons/import_beliefs.py` initializes each claim with `"source": ""` and then fills it from the markdown line:

```python
if line.startswith("- Source: "):
    current["source"] = line[len("- Source: "):].strip()
```

So an input like:

```markdown
### obs-1 [IN] OBSERVATION
The API uses REST endpoints.
- Source: repo/api.py
```

produces:

```python
claim["source"] == "repo/api.py"
```

The tests confirm this behavior: `tests/test_import_beliefs.py` expects `by_id["premise-a"]["source"] == "repo/file.md"`.

`import_into_network()` then uses that parsed claim data when creating nodes. The exact `add_node(...)` call is beyond the 200-line excerpt, but the parser and tests show that the source string is the bare path exactly as read from the `- Source:` line. No `agent:` or `expert:` prefix is added there.

`source_type` is separate metadata, not part of `source`. For example, `tests/test_source_type.py` adds a node with `source="repo/file.py"` and `source_type="code"` and verifies that `source_type` is stored under `metadata["source_type"]`.

### Agent import: source is preserved; agent identity goes elsewhere

For agent imports, `reasons/import_agent.py` normalizes both markdown and JSON inputs.

In `_normalize_markdown()`, the normalized claim includes:

```python
"source": c["source"]
```

In `_normalize_json()`, it includes:

```python
"source": ndata.get("source", "")
```

So agent import does **not** rewrite a belief’s source to something like `aap-expert:repo/file.md`. It preserves whatever `source` was in the source file.

Agent identity is encoded in other places:

- Synthetic agent kill-switch nodes use IDs like `{agent_name}:active` and `{agent_name}:inactive` in `_ensure_agent_nodes()` (`reasons/import_agent.py:41-65`).
- Those synthetic nodes get `metadata={"agent": agent_name, "role": "agent_premise"}` or `role: "agent_inactive"`.
- Their `source` is the `source_path` argument, not a generated agent-prefixed string.
- Labels may be generated as `f"imported from agent: {agent_name}"` (`reasons/import_agent.py:224`, `reasons/import_agent.py:233`) or `f"added by agent: {namespace}"` (`reasons/api.py:362`), but these are labels, not `source` fields.

So for:

```bash
reasons import-agent aap-expert ~/git/aap-expert/beliefs.md
```

the belief nodes imported from that file keep their own `source` values. The synthetic `aap-expert:active` node’s `source` is `~/git/aap-expert/beliefs.md` if that path is passed as `source_path`.

### Expert handling: not observed as a source-prefix rule

The grep results for `agent:|expert:` do not show any `source=` assignment constructing an `expert:` prefix. “Expert” appears in usage examples such as:

```text
reasons import-agent aap-expert ~/git/aap-expert/beliefs.md
reasons import-agent rhel-expert ~/git/rhel-expert/network.json
```

and possibly as part of agent names, but not as a distinct source-generation branch.

The observed code does not support a conclusion like:

```python
source = f"expert:{expert_name}:{path}"
```

If such behavior exists, it is outside the provided observations.

### `source` and `agent` are separate concepts

`reasons/check_stale.py` reinforces that `source` and `agent` are separate:

```python
source: str,
...
agent: str | None = None,
...
if not source:
```

and later:

```python
if not node.source:
```

So the staleness logic treats `source` as a source path/string and `agent` as separate contextual metadata. It does not expect agent identity to be embedded in the `source` field.

Also, API load paths preserve stored source strings directly:

- `source=ndata.get("source", "")` at `reasons/api.py:2171`
- `source=ndata.get("source", "")` at `reasons/api.py:2217`
- `source=ndata.get("source", "")` at `reasons/api.py:2473`
- `node.source = ndata.get("source", "")` at `reasons/api.py:2561`
- `node.source = ndata.get("source", "")` at `reasons/api.py:2632`

That indicates `source` is a persisted field, not a string regenerated from agent/expert context on load.

### What is missing

The observations do not include the remainder of `reasons/import_agent.py` after line 200, where the shared import logic actually inserts normalized claims into the network. The exact `add_node(... source=...)` calls for imported agent beliefs are not shown. However, the normalization functions clearly carry through the input `source` field, and no observed line constructs an `agent:` or `expert:` source prefix.

If you need absolute certainty about expert-specific source handling, inspect the rest of `reasons/import_agent.py` and search for any `source =` assignments involving `expert`.

## Topics to Explore

- [file] `reasons/import_agent.py` — The remainder after line 200 contains the shared insertion logic; verify whether it ever rewrites `source` during agent import.
- [function] `reasons/import_beliefs.py:import_into_network` — Shows how parsed `claim["source"]` is passed into node creation for normal belief imports.
- [function] `reasons/import_agent.py:_normalize_markdown` — Confirms that agent-imported markdown claims preserve their source field unchanged.
- [function] `reasons/check_stale.py:check_stale` — Clarifies how `source` and `agent` are used independently during staleness checks.
- [general] `expert-agent-naming` — Search for any code that treats agent names ending in `-expert` specially; current observations show only example names, not source-prefix behavior.

## Beliefs

- `import-beliefs-source-parsed-from-source-line` — `parse_beliefs` stores the literal text after `- Source: ` in `claim["source"]` without prefixing it.
- `agent-import-preserves-claim-source` — `_normalize_markdown` and `_normalize_json` carry through the input `source` value rather than synthesizing an agent-prefixed source.
- `agent-premise-source-is-source-path` — `_ensure_agent_nodes` assigns `source=source_path` to `agent:active` and `agent:inactive`, not `agent:<name>`.
- `check-stale-keeps-source-and-agent-separate` — `check_stale` accepts `source: str` and `agent: str | None` as separate parameters.
- `source-field-roundtrips-through-api-load` — API load paths set `node.source = ndata.get("source", "")`, preserving stored source strings.
