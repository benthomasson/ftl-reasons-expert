---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The `inactive` premise + justification-outlist mechanism; it is the hardest invariant to maintain through sync and is the reason `TestSyncAgentRevocation` exists.

# The `inactive` premise + justification-outlist mechanism

## The problem it solves

An imported agent contributes many beliefs. We want a **single kill switch** per agent: retracting one thing should logically retract *all* of that agent's beliefs in one cascading step — without, however, making every belief depend on the agent's "active" premise in a way that would prevent per-belief retraction later.

Standard TMS (Doyle) primitives only give us two levers:

- **Antecedents (inlist)** — a justification is valid only if *all* antecedents are IN.
- **Outlist** — a justification is valid only if *all* outlist nodes are OUT.

If you naively put `agent:active` in the *antecedents* of every imported belief, retracting `agent:active` would cascade OUT — good. But it would also mean every belief has an "always-valid fallback" tied to the premise, which defeats downstream per-belief retraction/dependency tricks the system relies on (see `reasons/import_agent.py` lines 11–15: **"not placed in antecedents … Instead, `agent:inactive` is placed in the outlist"**).

So the mechanism uses the *dual*:

## The relay pair

`_ensure_agent_nodes` (`reasons/import_agent.py:41`) creates two per-agent nodes:

1. **`{agent}:active`** — a plain *premise* node (no justifications). It is IN by default.
2. **`{agent}:inactive`** — a *relay* node with justification `SL(antecedents=[], outlist=["{agent}:active"])`.

Because the outlist requires all listed nodes to be OUT, `inactive` is:

| `active` state | `inactive` state |
|---|---|
| IN (default) | OUT |
| OUT (retracted) | **IN** |

That's the kill switch. When `active` is retracted (`inactive` goes IN), every belief whose justification has `inactive` in its outlist loses that justification and cascades OUT. The docstring at lines 5–11 spells this out: *"retracting agent:active makes agent:inactive go IN, which cascades OUT every belief from that agent via outlist."*

## Wiring it into every belief

`_build_justifications` (`reasons/import_agent.py:218`) is the choke point. Line 223:

```python
outlist = [inactive_id] + [f"{prefix}{o}" for o in rj["outlist"]]
```

The `inactive_id` is **prepended** to whatever outlist the source belief already declares. So a belief that says "believe X unless Y" becomes "believe X unless Y *or* the agent is revoked." This is the hard invariant:

> Every imported belief must carry `{agent}:inactive` in its outlist.

Two additional nuances flow from this:

- **Bare beliefs** (IN, no dependencies) get a synthetic justification `SL(antecedents=[], outlist=[inactive_id])` at line 232 so they still go OUT when the agent is revoked.
- **Dependents registration**: outlist nodes may be created after the beliefs that reference them (`_fixup_dependents` / `_rebuild_dependents`, `reasons/import_agent.py:29–38`; docs at line 46). The network's dependent-graph has to be rebuilt because `add_node` couldn't register the outlist edge at creation time. Forget this and revocation won't actually cascade — the graph won't *know* the belief depends on `inactive`.

The PostgreSQL backend mirrors this exactly to keep the two storage engines behaviorally identical:

- `reasons/pg.py:2542` builds `inactive_id = f"{agent_name}:inactive"`.
- Lines 2550/2554 insert the relay with its `active` outlist.
- Line 2587: `outlist = [inactive_id] + [...]` — same prepend in the pg path.
- Line 2596: the bare-belief `SL(antecedents=[], outlist=[inactive_id])` fallback.

So any change to how import builds justifications must be applied **twice** (SQLite/network path in `import_agent.py` and the Postgres path in `pg.py`) or the two stores will diverge.

## Why sync makes this the hardest invariant

`sync_agent` (`reasons/import_agent.py:564`, `sync_agent_json:585`; dispatch in `reasons/api.py:2016`; Postgres at `reasons/pg.py:2662`) doesn't just add new beliefs — it must:

- add new beliefs (with `inactive` in outlist),
- remove/retract beliefs (won't disturb the invariant),
- update text on existing beliefs (justifications unchanged — safe),
- **replace justifications** when dependencies/status change (`_update_node_justifications` at line ~85 replaces the list and calls `_rebuild_dependents`),
- flip IN↔OUT when the remote says so.

Every one of those paths has a chance to *recompute* the justification list and forget to re-prepend `inactive_id`, or to replace a justification list on a node without re-running `_build_justifications` and thus drop the kill switch. If a single belief loses `inactive` from its outlist, then retracting `active` no longer retracts that belief — the kill switch silently fails to reach every belief.

That is precisely what `TestSyncAgentRevocation` (`tests/test_sync_agent.py:472`) exists to lock down. It's the regression test that — after whatever combination of add/update/removal sync operations — verifies that revoking `agent:active` still cascades *all* of that agent's beliefs to OUT.

## Topics to Explore

- [file] `reasons/import_agent.py` — The full import/sync module; `_ensure_agent_nodes`, `_build_justifications`, `_update_node_justifications`, and both `sync_agent` variants are all here.
- [function] `reasons/import_agent.py:_build_justifications` — The single function that encodes the invariant; every belief's outlist gets `inactive_id` prepended here.
- [function] `tests/test_sync_agent.py:TestSyncAgentRevocation` — The regression test (line 472) proving revocation still cascades after a sync.
- [file] `reasons/pg.py` — The Postgres mirror of the same mechanism around lines 2542–2672; any invariant must be maintained in both places.
- [general] `network-rebuild-dependents` — Why `_fixup_dependents`/`_rebuild_dependents` exists: outlist edges added after node creation require explicit dependent re-registration or revocation won't actually cascade.

## Beliefs

- `active-premise-is-retraction-entry-point` — `{agent}:active` is a plain premise node whose retraction is the sole trigger for revoking an agent.
- `inactive-relay-inverts-active` — `{agent}:inactive` uses justification `SL(antecedents=[], outlist=[active])`, so it is IN exactly when `active` is OUT.
- `every-imported-belief-has-inactive-in-outlist` — `_build_justifications` prepends `{agent}:inactive` to every belief's outlist, and bare beliefs get a fallback `SL([], outlist=[inactive])`.
- `inactive-not-in-antecedents` — `inactive` is deliberately placed in outlist, never antecedents, to avoid an always-valid per-belief fallback.
- `sync-must-preserve-inactive-outlist` — `sync_agent` and its Postgres twin both re-prepend `inactive_id` on every justification rewrite; losing it silently breaks the kill switch, which is why `TestSyncAgentRevocation` exists.
- `inactive-mechanism-requires-rebuild-dependents` — outlist edges are registered only after `_rebuild_dependents()` (`import_agent.py:29–38`), since outlist nodes can be created after the beliefs that reference them.
