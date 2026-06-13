# agent

[Back to index](index.md)

In the ftl-reasons Truth Maintenance System, an **agent** is an external belief source whose nodes can be imported into a local TMS database while maintaining strict isolation guarantees. The agent subsystem provides complete lifecycle management — from initial import through ongoing synchronization — with layered containment mechanisms that prevent one agent's belief state from interfering with another's (agent-subsystem-is-self-contained).

## Import and Synchronization

Agents are introduced to the system through `import_agent`, which brings an external agent's beliefs into the local database. The `sync_agent` function provides an incremental update path: when called for an agent with no prior import, it behaves identically to `import_agent`, creating the `agent:active` premise and returning `created_premise: True` (sync-agent-first-sync-equals-import). This eliminates the need for a separate import step before synchronization can begin.

`sync_agent` follows remote-wins semantics (sync-agent-remote-wins) — the external agent's current state is authoritative. It returns a dictionary with keys `beliefs_added`, `beliefs_removed`, `beliefs_updated`, and `beliefs_unchanged`, giving callers precise accounting of what changed (sync-agent-returns-count-dict). The function also records the agent's source file path in the repo registry, enabling the system to track which file each agent's beliefs originated from (sync-registers-agent-repo-path).

### Idempotency and Safety

Calling `sync_agent` twice with identical data produces zero additions, removals, and updates on the second call — a no-op by design (sync-agent-is-idempotent). Crucially, sync preserves the agent's outlist-based justification structure: after sync completes, revoking `agent:active` still cascades all agent beliefs to OUT, proving that synchronization does not corrupt the kill-switch wiring (sync-agent-preserves-cascade-structure). Together, these properties make sync safe for automated reconciliation workflows.

## Isolation Mechanisms

Agent beliefs are protected by two independent containment mechanisms operating at different system levels (agent-isolation-spans-identity-and-authorization).

### Identity-Level Isolation

At the identity level, namespace prefixing prevents ID collisions between agents — each imported node is prefixed with the agent's name using colon-separated notation (namespace-prefix-is-agent-colon). Combined with the active/inactive relay pair, this provides per-agent kill-switch semantics: each agent's imported beliefs reference only their own `inactive` node in their outlist, so retracting one agent's active premise does not affect other agents' beliefs (agent-cascades-are-isolated-by-namespace). The import process creates two control nodes per agent — an `active` premise and an `inactive` relay — that together form the lifecycle control structure (agent-import-creates-two-control-nodes).

### Authorization-Level Isolation

At the authorization level, transitive subset-gated access tags control per-caller visibility with tag inheritance. This layer is independent of namespace isolation, meaning an agent's beliefs can be namespace-isolated yet still governed by fine-grained access policies (agent-isolation-spans-identity-and-authorization).

## Lifecycle and Defeat Reversibility

The agent subsystem handles the full lifecycle: import accommodates mixed truth states and topological cycles, namespace/relay pairs provide isolation and kill-switches, and all defeat operations are reversible for agent reactivation (agent-subsystem-is-self-contained). This reversibility is significant — deactivating an agent via its kill-switch cascades all its beliefs to OUT, but the structure remains intact so that reasserting the active premise restores the agent's beliefs without reimporting.
