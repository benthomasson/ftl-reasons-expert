# agent

[Back to index](index.md)

The agent subsystem enables multi-agent federation by allowing one truth maintenance system to import, isolate, and synchronize beliefs from external agents. Each agent's beliefs are namespaced, controlled by a dedicated relay pair, and participate in the same justification mechanics as locally-created beliefs.

## Import and Control Nodes

When an agent's beliefs are first brought into the system, the import process creates a two-node control structure: an `active` premise and an `inactive` relay node, linked as a pair (`agent-import-creates-two-control-nodes`). The `active` node serves as a kill-switch — retracting it cascades all of that agent's imported beliefs to OUT, while the `inactive` node provides the complementary signal for outlist-based defeat. This relay pair is the foundation of per-agent lifecycle control.

The `sync_agent` function unifies import and update into a single entry point. When no prior import exists for an agent, `sync_agent` behaves identically to `import_agent` — creating the `agent:active` premise, wiring namespace prefixes, and returning `created_premise: True` — rather than requiring a separate import step (`sync-agent-first-sync-equals-import`, `sync-agent-first-call-equals-import`).

## Namespace Isolation

Agent beliefs are doubly isolated. Namespace prefixing prevents ID collisions between agents, using a colon-separated format (`namespace-prefix-is-agent-colon`). The active/inactive relay pair then provides per-agent kill-switch semantics without cross-agent interference (`agent-isolation-through-namespace-and-relay`). Concretely, retracting one agent's active premise does not affect other agents' beliefs, because each agent's imported beliefs reference only their own `inactive` node in their outlist (`agent-cascades-are-isolated-by-namespace`).

Beyond identity-level isolation, the system provides a second, independent containment layer: transitive subset-gated access tags control per-caller visibility with tag inheritance, providing authorization-level isolation (`agent-isolation-spans-identity-and-authorization`). These two mechanisms — namespace/relay for identity, access tags for authorization — operate at different system levels and reinforce each other.

## Sync Semantics

The `sync_agent` operation follows remote-wins semantics (`sync-agent-remote-wins`), meaning the remote agent's current state is authoritative. After sync completes, it returns a dictionary with keys `beliefs_added`, `beliefs_removed`, `beliefs_updated`, and `beliefs_unchanged` for precise accounting of what changed (`sync-agent-returns-count-dict`).

Sync is idempotent: two consecutive calls with identical file content produce zero additions, removals, and updates on the second call (`sync-agent-idempotent`, `sync-agent-is-idempotent`). Crucially, sync preserves the agent's outlist-based justification structure — after sync completes, revoking `agent:active` still cascades all agent beliefs to OUT, proving that sync does not corrupt kill-switch wiring (`sync-agent-preserves-cascade-structure`).

The system also records the agent's source file path in the repo registry via `list_repos`, enabling tracking of which file each agent's beliefs originated from (`sync-registers-agent-repo-path`).

## Lifecycle Management

The agent subsystem is self-contained: import handles mixed truth states and topological cycles, namespace/relay pairs provide isolation and kill-switches, and all defeat operations are reversible for agent reactivation (`agent-subsystem-is-self-contained`). This reversibility is key — an agent can be deactivated by retracting its `active` premise, then later restored by reasserting it, with all dependent beliefs cascading back to IN.

## Revision and Multi-Agent Safety

Several higher-level beliefs about multi-agent safety have been retracted (OUT), reflecting refinements in the network's understanding. The claim that agent-imported beliefs participate in the full revision system — undergoing the same outlist defeat and contradiction resolution as local beliefs — is currently OUT (`agent-beliefs-undergo-full-revision`), as is the broader claim that multi-agent reasoning is sound and scalable (`multi-agent-reasoning-is-sound-and-scalable`). The assertion that multi-agent operation is semantically uniform with local revision (`multi-agent-revision-is-semantically-uniform`) and that safety spans all architectural layers (`multi-agent-safety-spans-all-layers`) are likewise OUT. These retractions propagated from upstream dependencies rather than from direct refutation of agent mechanics — the foundational isolation and lifecycle beliefs remain IN.
