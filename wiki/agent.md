# agent

[Back to index](index.md)

### agent-beliefs-undergo-full-revision
**Status:** OUT

Agent-imported beliefs participate in the full revision system: the self-contained agent subsystem provides isolated lifecycle management and reversible defeat, while the comprehensive revision system ensures agent beliefs are subject to the same outlist defeat and contradiction resolution as locally-created beliefs — no revision exception exists for external provenance.

**Depends on:** [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained), [belief-revision-is-comprehensive-and-minimal](revision.md#belief-revision-is-comprehensive-and-minimal)
**Supports:** [all-belief-origins-share-deterministic-revision](deterministic.md#all-belief-origins-share-deterministic-revision)

### agent-cascades-are-isolated-by-namespace
**Status:** IN

Retracting one agent's active premise does not affect other agents' beliefs, because each agent's imported beliefs reference only their own `inactive` node in their outlist

**Supports:** [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay)

### agent-import-creates-two-control-nodes
**Status:** IN

Covered by existing `active-inactive-relay-pair` which captures the same two-node control structure


### agent-isolation-spans-identity-and-authorization
**Status:** IN

Agent beliefs are isolated through two independent containment mechanisms at different system levels: namespace prefixing with relay pairs provides identity-level isolation (preventing ID collisions and enabling per-agent lifecycle control via kill-switch), while transitive subset-gated access tags provide authorization-level isolation (controlling per-caller visibility with tag inheritance).

**Depends on:** [access-control-is-transitive-subset-gated](other.md#access-control-is-transitive-subset-gated), [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay)

### agent-isolation-through-namespace-and-relay
**Status:** IN

Agent beliefs are doubly isolated: namespace prefixing prevents ID collisions, while the active/inactive relay pair provides per-agent kill-switch semantics without cross-agent interference

**Depends on:** [active-inactive-relay-pair](other.md#active-inactive-relay-pair), [active-not-in-antecedents](other.md#active-not-in-antecedents), [agent-cascades-are-isolated-by-namespace](agent.md#agent-cascades-are-isolated-by-namespace), [import-agent-namespace-prefix](import.md#import-agent-namespace-prefix)
**Supports:** [agent-isolation-spans-identity-and-authorization](agent.md#agent-isolation-spans-identity-and-authorization), [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained)

### agent-subsystem-is-self-contained
**Status:** IN

The agent subsystem provides complete lifecycle management: import handles mixed truth states and topological cycles, namespace/relay pairs provide isolation and kill-switches, and all defeat operations are reversible for agent reactivation.

**Depends on:** [agent-isolation-through-namespace-and-relay](agent.md#agent-isolation-through-namespace-and-relay), [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [import-handles-heterogeneous-truth-states](import.md#import-handles-heterogeneous-truth-states)
**Supports:** [agent-beliefs-undergo-full-revision](agent.md#agent-beliefs-undergo-full-revision), [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained), [multi-agent-safety-spans-all-layers](agent.md#multi-agent-safety-spans-all-layers)

### multi-agent-reasoning-is-sound-and-scalable
**Status:** OUT

The system provides both individually sound reasoning (deterministic, reversible, terminating truth maintenance) and safe multi-agent operation (isolated namespaces, reversible lifecycle control, clean architectural boundaries), enabling arbitrarily many agents without sacrificing correctness guarantees.

**Depends on:** [multi-agent-safety-spans-all-layers](agent.md#multi-agent-safety-spans-all-layers), [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible)
**Supports:** [extensions-compose-transparently-on-core](other.md#extensions-compose-transparently-on-core), [multi-agent-revision-is-semantically-uniform](agent.md#multi-agent-revision-is-semantically-uniform), [system-is-minimal-sound-and-scalable](system.md#system-is-minimal-sound-and-scalable)

### multi-agent-revision-is-semantically-uniform
**Status:** OUT

Multi-agent operation does not carve out exceptions to the universal revision semantics — agent beliefs undergo the same uniform revision (outlist defeat, contradiction backtracking, edge-case handling) as local beliefs because agent namespacing and relay pairs operate above the evaluation layer, not within it.

**Depends on:** [belief-revision-covers-all-cases-uniformly](revision.md#belief-revision-covers-all-cases-uniformly), [multi-agent-reasoning-is-sound-and-scalable](agent.md#multi-agent-reasoning-is-sound-and-scalable)
**Supports:** [all-mutation-sources-are-safe-and-uniform](safe.md#all-mutation-sources-are-safe-and-uniform), [complete-operational-uniformity-across-all-sources](complete.md#complete-operational-uniformity-across-all-sources)

### multi-agent-safety-spans-all-layers
**Status:** OUT

Multi-agent operation is safe across the full system: agent isolation prevents cross-contamination between namespaces with reversible lifecycle control, while data integrity is enforced across all three architectural layers through clean boundaries and snapshot persistence.

**Depends on:** [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained), [data-integrity-spans-architecture](spans.md#data-integrity-spans-architecture)
**Supports:** [all-external-belief-paths-are-safely-bounded](external.md#all-external-belief-paths-are-safely-bounded), [all-external-inputs-produce-correct-state](external.md#all-external-inputs-produce-correct-state), [multi-agent-reasoning-is-sound-and-scalable](agent.md#multi-agent-reasoning-is-sound-and-scalable)

### namespace-prefix-is-agent-colon
**Status:** IN

Covered by existing `namespace-prefix-is-colon-separated` and `import-agent-namespace-prefix`


### sync-agent-first-call-equals-import
**Status:** IN

Calling `sync_agent` for an agent with no prior import produces identical results to `import_agent`, including all node creation and namespace wiring


### sync-agent-first-sync-equals-import
**Status:** IN

When no prior import exists for an agent, `sync_agent` behaves identically to `import_agent` — creating the `agent:active` premise and returning `created_premise: True` — rather than requiring a separate import step.


### sync-agent-idempotent
**Status:** IN

Two consecutive `sync_agent` calls with identical file content produce zero additions, removals, and updates on the second call

**Supports:** [critical-operations-converge-to-fixed-points](other.md#critical-operations-converge-to-fixed-points)

### sync-agent-is-idempotent
**Status:** IN

Calling `sync_agent` twice with identical data produces the same database state; the second call returns zero adds, removes, and updates — a no-op by design.

**Supports:** [sync-is-safe-for-automated-reconciliation](safe.md#sync-is-safe-for-automated-reconciliation)

### sync-agent-preserves-cascade-structure
**Status:** IN

After `sync_agent` completes, the agent's outlist-based justification structure remains intact — revoking `agent:active` still cascades all agent beliefs to OUT, proving sync does not corrupt kill-switch wiring.

**Supports:** [sync-is-safe-for-automated-reconciliation](safe.md#sync-is-safe-for-automated-reconciliation)

### sync-agent-remote-wins
**Status:** IN

Duplicates existing belief `sync-is-remote-wins`


### sync-agent-returns-count-dict
**Status:** IN

`sync_agent` returns a dict with keys `beliefs_added`, `beliefs_removed`, `beliefs_updated`, and `beliefs_unchanged` for accurate accounting of what changed


### sync-registers-agent-repo-path
**Status:** IN

`sync_agent` records the agent's source file path in the repo registry (`list_repos`), enabling the system to track which file each agent's beliefs originated from.

