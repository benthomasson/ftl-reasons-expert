---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _detect_agents in reasons/derive.py

# `_detect_agents` — Explanation

**Location:** `reasons/derive.py`

```python
def _detect_agents(nodes):
    agents = defaultdict(list)
    for nid, node in nodes.items():
        if ":" in nid:
            agent = nid.split(":")[0]
            # Skip the :active premise nodes
            if nid.endswith(":active"):
                continue
            agents[agent].append(nid)
    return dict(agents)
```

---

## 1. Purpose

`_detect_agents` scans the full belief-network node map and groups node IDs by their **agent namespace prefix**. It exists so the derive pipeline knows whether the network is a single-agent (local) network or a **multi-agent / `import-agent` network**, and — if the latter — which agents are present and how many nodes each contributes.

The output directly feeds:
- `domain_context` text in the derive prompt (listing agents),
- the `CROSS_AGENT_TASK` section that instructs the LLM to propose cross-agent derivations,
- per-agent budget allocation inside `_build_beliefs_section`,
- per-agent counts in the `agents_stats` block of the prompt and in the returned `stats` dict.

Agent-namespaced node IDs follow the convention `agent-name:belief-id`. A second convention — node IDs ending in `:active` — is treated as an internal **premise/sentinel node** (created by `import-agent` to represent the agent's active state) and is deliberately excluded from agent grouping.

---

## 2. Contract

**Preconditions**
- `nodes` is a `dict`-like mapping of `node_id: str → node_data: dict` (as produced by `export_network` / `api`).
- `node_id` is a string. Nodes with a `:` in the ID are considered agent-namespaced.
- Agent namespaces use the convention `<agent-name>:<belief-id>`; the "active premise" node uses the suffix `:active`.

**Postconditions**
- Returns a plain `dict[str, list[str]]` (not the internal `defaultdict`), where each key is an agent name and each value is the list of that agent's non-`:active` node IDs.
- Nodes without `:` in their ID (local/root beliefs) are **not** returned.
- Nodes whose ID ends in `:active` are **not** returned **and do not create an entry** for their agent (because `continue` runs before `agents[agent].append(...)`).
- List values preserve the iteration order of `nodes.items()` (Python 3.7+ insertion order).

**Invariants**
- No caller-visible mutation of `nodes`.
- Every returned key corresponds to at least one returned node ID (no empty-list entries).
- The function never consults `node["agent"]` even though the docstring mentions it.

---

## 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `nodes` | `dict[str, dict]` | Full node map from `export_network`. Only keys are inspected. | Empty dict → returns `{}`. Non-dict with `.items()` still works. Passing `None` raises `AttributeError` on `.items()`. Node values are ignored entirely — a `ValueError` or `TypeError` in the value dict cannot cause a failure here. |

**Edge cases of the ID format**
- `"plain-id"` → skipped (no `:`).
- `"agent:belief"` → grouped under `"agent"`.
- `"agent:sub:belief"` → grouped under `"agent"` (split on first colon only; remainder retained).
- `"foo:active"` → skipped.
- `"foo:bar:active"` → skipped (any `:active` suffix skips the node — this is a subtle behavior; in practice IDs are one-colon deep).
- `":foo"` → grouped under `""` (empty-string agent). Not filtered out; the code assumes this doesn't happen.
- `"config:file:path"` → would be misclassified as agent `"config"`. The function has **no validation** that `:` in an ID truly indicates an agent namespace.

---

## 4. Return Value

A `dict[str, list[str]]`:

```python
{
    "agent-a": ["agent-a:belief-1", "agent-a:belief-2"],
    "agent-b": ["agent-b:belief-1"],
}
```

- Empty dict if the network is local-only (no colon IDs) or all colon IDs end in `:active`.
- Callers must guard on truthiness (`if agents:`) before treating the network as multi-agent — the callers in this file do exactly that.
- Because the return is a **plain** `dict` (via `dict(agents)`), a caller that later writes `result[unknown] = ...` will raise `KeyError`, not silently grow. The conversion also decouples the result from the internal `defaultdict`, so mutating the result does not affect internal state.

---

## 5. Algorithm

1. `agents = defaultdict(list)` — accumulator.
2. Iterate `nodes.items()` in insertion order.
3. For each `nid` that contains `:`, split on the first colon to obtain `agent`.
4. If `nid.endswith(":active")`, skip this node entirely (note: `agent` has already been computed but is unused — the skip happens before any key is created in `agents`).
5. Otherwise, `agents[agent].append(nid)` — this creates a list entry for a new agent on demand.
6. Return `dict(agents)` — a plain dict copy.

The implementation is O(N) in node count and O(K) allocations for lists (one per agent).

---

## 6. Side Effects

**None.** No I/O, no mutation of `nodes`, no global state, no external calls, no randomness. It's a pure function of its input.

---

## 7. Error Handling

- **No exceptions raised** by this function for well-formed input.
- If `nodes` is not dict-like, `AttributeError` bubbles up from `.items()`.
- If `nid` is not a `str`, `TypeError` bubbles up from the `in` / `split` / `endswith` operations.
- The function does not swallow errors or return sentinel values; a `None` or non-string ID will crash. The upstream `export_network` contract is the only guarantee that IDs are strings.

---

## 8. Usage Patterns

Primary caller: `build_prompt` in the same module.

```python
agents = _detect_agents(nodes)

# Domain context
if agents:
    agent_list = ", ".join(sorted(agents.keys()))
    domain_context = f"This network contains beliefs from multiple agents: {agent_list}. ..."

# Cross-agent task instructions
cross_agent_task = CROSS_AGENT_TASK if agents else ""

# Per-agent stats block
if agents:
    parts = [f"- Agents: {len(agents)}"]
    for name in sorted(agents):
        parts.append(f"  - {name}: {len(agents[name])} beliefs")
    agents_stats = "\n".join(parts)

# Budget allocation
_build_beliefs_section(nodes, derived, agents, ...)

# Emitted in stats
stats["agents"] = len(agents)
stats["agent_names"] = sorted(agents.keys()) if agents else []
```

Caller obligations:
- Guard on `if agents:` — an empty dict means "local network, no cross-agent task".
- Do not rely on `agents` reflecting **IN-only** counts. `_detect_agents` returns every colon-namespaced node regardless of `truth_value`. `_build_beliefs_section` later re-derives per-agent counts by intersecting with `in_nodes`; the `agents_stats` block in the prompt uses the raw `len(agents[name])`, so the prompt can display a per-agent total (IN + OUT) while the beliefs section shows only the IN subset. Minor consistency quirk.
- Agent names are treated as opaque strings (used in `startswith(f"{name}:")` in `_build_beliefs_section`), so **no colon in an agent name is permissible** — otherwise all subsequent lookups would mis-segment.

---

## 9. Dependencies

- `collections.defaultdict` — imported at module top level in `reasons/derive.py`.
- No other internal or external dependencies. No imports from `reasons.api` or related modules.

---

## Assumptions Not Enforced by the Type System

1. **`node_id` is a `str`.** The `in`, `.split`, and `.endswith` calls will raise `TypeError` on non-strings. Nothing checks.
2. **`:` exclusively marks agent namespacing.** Any present or future ID convention using `:` outside agent namespacing (e.g., a URI scheme, a `config:file:path` label) would be silently mis-grouped.
3. **Agent names contain no `:`.** Enforced implicitly by `_build_beliefs_section`'s `startswith(f"{name}:")` check — an agent name with a colon would silently mismatch there but still pass here.
4. **`:active` suffix uniquely identifies `import-agent`'s premise node.** Any legitimate belief ID ending in `:active` is dropped without indication.
5. **`node["agent"]` metadata is irrelevant here.** The docstring claims it exists; the code never reads it. Metadata-based agent attribution is either handled elsewhere (in `import-agent`) or the docstring is stale.

---

## Topics to Explore

- [file] `reasons/derive.py` — Full derive pipeline; see how the `agents` dict flows into `_build_beliefs_section` and the prompt template.
- [function] `reasons/derive.py:build_prompt` — The only caller; shows domain context construction, stats assembly, and template placeholders (`agents_stats`, `cross_agent_task`).
- [function] `reasons/derive.py:_build_beliefs_section` — Re-derives per-agent belief sets against `in_nodes` for budget allocation; explains the IN/OUT count mismatch noted above.
- [general] `import-agent-workflow` — Where the `agent-name:belief-id` and `:active` conventions originate (likely in an importer module or CLI subcommand).
- [general] `active-premise-nodes` — What the `:active` sentinel node represents semantically and whether any downstream code depends on its presence.

---

## Beliefs

- `detect-agents-returns-dict-of-lists` — `_detect_agents` returns a plain `dict[str, list[str]]` mapping agent name to that agent's node IDs.
- `detect-agents-agent-name-is-first-colon-segment` — The agent name is `nid.split(":")[0]`, i.e. everything before the first colon.
- `detect-agents-skips-active-suffix` — Any node ID ending with `:active` is excluded from the output, and its agent is not added to the result if no other nodes exist for that agent.
- `detect-agents-ignores-node-metadata` — The function inspects only node IDs; the `agent` metadata field referenced in the docstring is never consulted.
- `detect-agents-counts-all-truth-values` — The result includes IN and OUT nodes alike; callers such as `_build_beliefs_section` must intersect with `in_nodes` for IN-only statistics.
