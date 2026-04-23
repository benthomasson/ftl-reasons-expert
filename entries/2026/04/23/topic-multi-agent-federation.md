# Topic: How agents are namespaced, the active/inactive relay pattern, and kill-switch cascades

**Date:** 2026-04-23
**Time:** 16:46

Now I have a thorough understanding of all three patterns. Here's the explanation:

---

# Agent Namespacing, Active/Inactive Relay, and Kill-Switch Cascades

## 1. Agent Namespacing

Agents in ftl-reasons are namespaced using a **colon-separated prefix** scheme. When an agent named `rhel-expert` is imported, every belief ID gets prefixed: `alpha-fact` becomes `rhel-expert:alpha-fact`.

The prefix is constructed at `import_agent.py:292`:
```python
prefix = f"{agent_name}:"
```

And applied to every imported belief at `import_agent.py:305`:
```python
node_id = f"{prefix}{claim['id']}"
```

This also applies to dependency references — antecedents and outlist entries are prefixed so they point to the namespaced copies (`import_agent.py:249-250`):
```python
antecedents = [f"{prefix}{a}" for a in rj["antecedents"]]
outlist = [inactive_id] + [f"{prefix}{o}" for o in rj["outlist"]]
```

Cross-namespace references are handled by `api.py:20-28` — the `_resolve_namespace()` function skips prefixing any ID that already contains a colon, so `other-agent:some-fact` passes through unchanged.

Namespaces are discovered at runtime by scanning for nodes matching the pattern `*:active` with `role: agent_premise` metadata (`api.py:87-113`).

## 2. The Active/Inactive Relay Pattern

Each imported agent gets exactly **two infrastructure nodes**, created by `_ensure_agent_nodes()` at `import_agent.py:45-72`:

| Node | Type | Default State | Purpose |
|------|------|---------------|---------|
| `agent:active` | Premise (no justifications) | **IN** | The trust signal — "this agent's beliefs are trusted" |
| `agent:inactive` | Derived (SL justification) | **OUT** | The kill switch — goes IN only when `active` is OUT |

The relay mechanism is the `inactive` node's justification (`import_agent.py:67`):

```python
Justification(type="SL", antecedents=[], outlist=[active_id])
```

This reads: "`inactive` is justified when `active` is OUT." Since `active` starts IN, `inactive` starts OUT.

Every imported belief includes `inactive_id` in its outlist (`import_agent.py:250`). The justification validity check at `network.py:717-732` requires **all outlist nodes to be OUT** for a justification to hold. So in normal operation, with `inactive` OUT, all beliefs can be IN.

A critical design decision is documented at `import_agent.py:9-11`: the `active` premise is **not** placed in antecedents. If it were, it would provide a second always-valid justification path, defeating per-belief retraction (you couldn't retract individual beliefs because `active` being IN would always re-validate them).

## 3. Kill-Switch Cascades

When you retract `agent:active`, a cascade propagates through the network via BFS (`network.py:675-702`):

**Step 1 — Retract the premise.** `network.retract("agent:active")` sets `active` to OUT and calls `_propagate()` (`network.py:76-102`).

**Step 2 — Relay flips.** The propagation checks `active`'s dependents. `inactive` depends on `active` via outlist. With `active` now OUT, `inactive`'s justification becomes valid (empty antecedents = trivially satisfied, outlist `[active]` = all OUT). `inactive` flips to IN.

**Step 3 — Beliefs cascade OUT.** Every imported belief has `inactive` in its outlist. With `inactive` now IN, every belief's justification is invalid. They all flip to OUT.

**Step 4 — Transitive cascade.** If belief B depends on belief A as an antecedent, and A just went OUT, then B also goes OUT — even if B belongs to a different agent. The BFS continues until no more truth values change.

The validity check that drives all of this is at `network.py:723-732`:

```python
inlist_ok = all(a in self.nodes and self.nodes[a].truth_value == "IN"
                for a in j.antecedents)
outlist_ok = all(o not in self.nodes or self.nodes[o].truth_value == "OUT"
                for o in j.outlist)
return inlist_ok and outlist_ok
```

### Cascade Isolation

Namespacing ensures agent kill-switches are independent. Retracting `agent-a:active` only makes `agent-a:inactive` go IN, which only affects beliefs with `agent-a:inactive` in their outlist. `agent-b`'s beliefs are untouched because they reference `agent-b:inactive` instead. This is tested explicitly at `tests/test_import_agent.py:149-168`.

### Restoration

The cascade is fully reversible. Calling `assert_node("agent:active")` (`network.py:104-123`) sets `active` back to IN, which makes `inactive` go OUT again, which re-validates all the agent's beliefs. The test at `test_import_agent.py:120-127` confirms that `alpha-fact` and `beta-depends-alpha` both come back.

### Import vs. Sync

Two modes exist for bringing in agent beliefs:

- **Import** (`import_agent.py:290-353`) — One-time load. Existing nodes are skipped.
- **Sync** (`import_agent.py:356-502`) — Remote-wins update. Text, justifications, and truth values are updated; beliefs removed from the remote are retracted locally; beliefs the remote restored (IN after being OUT) are re-asserted.

---

## Topics to Explore

- [function] `reasons_lib/network.py:recompute_all` — Full-network truth maintenance that runs after import/sync to catch transitive effects
- [file] `reasons_lib/storage.py` — How the node graph persists to SQLite, including the FTS5 index and propagation log
- [function] `reasons_lib/api.py:what_if_retract` — Simulates a retraction without committing it, useful for understanding cascade blast radius before pulling the trigger
- [file] `tests/test_sync_agent.py` — Covers the remote-wins merge semantics: how conflicts resolve, what happens when the remote removes or restores beliefs
- [general] `cross-namespace-dependencies` — What happens when agent-a's beliefs depend on agent-b's beliefs and agent-b is killed; how `_resolve_namespace` enables these cross-agent references

## Beliefs

- `namespace-prefix-is-colon-separated` — Agent namespacing uses the format `agent_name:belief_id`; a colon in a node ID signals it is already namespaced and `_resolve_namespace` will not double-prefix it
- `inactive-node-uses-outlist-not-antecedents` — The `agent:inactive` relay node's justification uses `outlist=[active_id]` with empty antecedents, not the reverse; this is a deliberate fix (documented at `import_agent.py:9-11`) that preserves per-belief retraction semantics
- `kill-switch-cascade-is-reversible` — Retracting `agent:active` cascades all agent beliefs to OUT, and re-asserting `agent:active` restores them to IN via the same BFS propagation
- `agent-cascades-are-isolated-by-namespace` — Retracting one agent's active premise does not affect other agents' beliefs, because each agent's beliefs reference only their own `inactive` node in their outlist
- `justification-validity-requires-inlist-in-and-outlist-out` — A justification is valid iff all antecedents are IN and all outlist nodes are OUT (`network.py:723-732`); this single rule drives both retraction cascades and kill-switch behavior

