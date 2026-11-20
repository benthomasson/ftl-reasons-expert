---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: add_justification in reasons/api.py

# `add_justification` — Functional Python API

## Purpose

`add_justification` appends a new **justification** (a support/defeat rule) to an **existing node** in the Reason Maintenance System. Nodes in an RMS can carry multiple justifications; this function adds one more without touching the node's text or metadata.

It exists as part of the flat "functional API" layer that lets CLI commands, LangGraph tools, and scripts operate on the network without ever opening a `Storage` lifecycle themselves — each function loads, mutates, saves, and returns JSON-serializable dicts.

It supports three justification shapes:
- **SL justification** (`sl="a,b"`) — conjunctive support: node is IN when *all* antecedents are IN and *no* outlist entry is IN.
- **CP justification** (`cp="a,b"`) — conditional-proof style support (type tag `"CP"`).
- **Outlist-only justification** (`unless="blocker"` with no `sl`/`cp`) — an SL with no positive antecedents; useful for premises that hold unless some other belief is IN.

It also supports an `any_mode` for disjunctive (OR) support: `sl="a,b,c"` becomes three separate single-antecedent SL justifications, one per antecedent.

---

## Contract

**Preconditions (not enforced by the type system):**
- Exactly one of `sl`, `cp`, or `unless` should carry the "meaningful" payload. Only the first non-empty one of `{sl, cp, unless}` is honored (see `sl` taking precedence over `cp` below). At least one must be non-empty or the function raises.
- The `node_id` is expected to already exist. Existence is only verified deep inside `net.add_justification` — this function does not pre-check.
- When `pg_conninfo` is supplied, `any_mode` must be `False`.

**Postconditions:**
- The (possibly namespace-prefixed) node now has one additional `Justification` in its `justifications` list.
- The network has been recomputed and persisted to the database (`_with_network(..., write=True)` saves on clean `__exit__`).
- Pub/sub change events are published if there are subscribers.

**Invariants:**
- `any_mode` only expands when `jtype == "SL"` **and** `len(antecedents) > 1`; otherwise it silently falls through to the normal single-justification path.
- On the PostgreSQL path, no `any_mode` support at all.

---

## Parameters

| Param | Meaning |
|---|---|
| `node_id: str` | Target node. If `namespace` is set and `node_id` has no `":"`, it is prefixed to `"{namespace}:{node_id}"`. |
| `sl: str = ""` | Comma-separated antecedent IDs for an SL justification. Empty string means "not provided". |
| `cp: str = ""` | Comma-separated antecedent IDs for a CP justification. **Ignored if `sl` is also non-empty.** |
| `unless: str = ""` | Comma-separated outlist IDs. Split, stripped, and **empty entries filtered out** (unlike `sl`/`cp`). Must be OUT for the justification to be satisfiable. |
| `label: str = ""` | Free-text label attached to the justification (used in explains, reviews, and inspection). |
| `namespace: str \| None = None` | If set, prefixes `node_id` and all antecedent/outlist IDs with `"{namespace}:"` — but **only for IDs not already containing a `":"`** (see `_resolve_namespace`). |
| `any_mode: bool = False` | Only meaningful for `sl` with multiple antecedents. Expands to one single-antecedent SL justification per antecedent (OR semantics). Ignored (misleadingly) for `cp` or `unless`. |
| `db_path: str = DEFAULT_DB` | SQLite database path, defaults to `reasons.db`. |
| `pg_conninfo, domain_id` | PostgreSQL backend override; presence of `pg_conninfo` routes to `PgApi`. |

**Edge case gaps in the types:**
- `sl`-split antecedents are **not** filtered for empty strings (`[a.strip() for a in sl.split(",")]`), whereas `outlist` is. So `sl=",a"` would produce `["", "a"]`.
- Providing **both** `sl` and `cp` silently drops `cp` (elif chain).
- Providing `any_mode=True, cp="a,b"` silently does *not* expand — `any_mode` only affects SL.

---

## Return Value

Returns a `dict` with at least:

```
{
  "node_id": str,
  "old_truth_value": "IN" | "OUT",
  "new_truth_value": "IN" | "OUT",
  "changed": list[str],          # node IDs whose truth value flipped
  "premise_count": int,          # wrapper-added field
}
```

Underlying fields come from `Network.add_justification`. The wrapper:
- Adds `premise_count = len(antecedents)` in the normal branch.
- Adds `premise_count = 1` in the `any_mode` branch — even though multiple justifications were created. This is arguably misleading to callers.
- In `any_mode`, only the **last** iteration's `result` is returned — the intermediate truth-value transitions from earlier per-antecedent calls are discarded.

**Caller responsibilities:**
- Wrap in try/except for `ValueError` (missing `sl`/`cp`/`unless`) and `KeyError` (node not found, raised by `Network.add_justification`).
- If tracking `changed` for pub/sub or UI refresh, only the final `changed` set is returned in `any_mode`.

---

## Algorithm

**Step 1 — Backend dispatch.** If `pg_conninfo` is present: raise `NotImplementedError` if `any_mode`, else `_pg_dispatch` to `PgApi.add_justification` and return.

**Step 2 — Parse outlist.** `outlist = [o.strip() for o in unless.split(",") if o.strip()]` (empty entries filtered).

**Step 3 — Select justification type / antecedents (priority: sl > cp > outlist-only):**
- `sl` non-empty → `antecedents = [a.strip() for a in sl.split(",")]`, `jtype = "SL"`
- `elif cp` non-empty → same for `cp`, `jtype = "CP"`
- `elif outlist` non-empty → `antecedents = []`, `jtype = "SL"`
- `else` → `raise ValueError("Must provide --sl, --cp, or --unless")` (message references CLI flags)

**Step 4 — Open write context.** `with _with_network(db_path, write=True) as net:` — always loads a fresh `Network` (never reuses the read cache for writes).

**Step 5 — Namespace resolution.** If `namespace`:
- Prefix `node_id`, each antecedent, each outlist ID via `_resolve_namespace`, which only prefixes strings without a `":"`.
- **Note:** unlike `add_node`, this does *not* inject `{namespace}:active` into the antecedents and does *not* ensure the namespace premise exists.

**Step 6a — `any_mode` expansion (SL only).**
- Loop over `antecedents`, creating one `Justification(type="SL", antecedents=[a], outlist=outlist, label=label)` per entry.
- Call `net.add_justification` on each.
- Return the last result with `premise_count = 1`.

**Step 6b — Single justification path.**
- Build one `Justification(type=jtype, antecedents=antecedents, outlist=outlist, label=label)`.
- `result = net.add_justification(node_id, justification)`.
- Set `result["premise_count"] = len(antecedents)`.
- Return.

**Step 7 — Persist.** `_with_network.__exit__` calls `self.store.save(self.network)` and publishes pub/sub change events *only if* no exception was raised. On exception, no save.

---

## Side Effects

- **Database write.** Full network is re-serialized on clean exit.
- **Truth-value propagation.** `Network.add_justification` recomputes the network; dependent nodes can flip IN/OUT.
- **Pub/sub.** If any subscriber is registered for `db_path`, in-flight `AND` deltas are computed and published.
- **Cache mutation.** On clean exit, the in-memory network cache for `db_path` is updated with the freshly mutated network (with the new file mtime).
- **No filesystem I/O beyond the DB.** Labels or text are not written.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| `sl`, `cp`, `unless` all empty | `ValueError("Must provide --sl, --cp, or --unless")`. |
| `pg_conninfo` set + `any_mode=True` | `NotImplementedError("any_mode is not supported with PostgreSQL")`. |
| `node_id` not found | Propagates whatever `Network.add_justification` raises (typically `KeyError`). No pre-check. |
| Invalid antecedent IDs in `sl`/`cp` | Not validated here; behavior depends on `Network.add_justification` (may add dangling dependents — see `test_dangling_dependents`). |
| Any exception inside the write context | `_with_network.__exit__` **does not save**; cache is still updated to the (possibly half-mutated) in-memory network. **The write may be silently discarded.** |

**Silent failures the type system doesn't catch:**
- Providing both `sl` and `cp` — only `sl` is used, no error.
- Providing `any_mode=True` with `cp` — mode silently ignored.
- Empty strings in `sl` antecedents (unfiltered, unlike outlist).

---

## Usage Patterns

**Typical CLI path — augment a derived belief with a second supporting rule:**
```
add_justification("claim-42", sl="doc-a,doc-b", label="cross-source support")
```

**Disjunctive support (OR):**
```
add_justification("claim-42", sl="evidence-1,evidence-2", any_mode=True)
# ⇒ two separate single-antecedent SL justifications; premise_count=1 on return
```

**Namespace-scoped belief:**
```
add_justification("claim-42", sl="src-1", namespace="agent-x")
# resolves to agent-x:claim-42 justified by agent-x:src-1
```

**Caller obligations:**
- Wrap in try/except `KeyError`/`ValueError`.
- For `any_mode`, do not rely on `changed` to reflect intermediate per-antecedent transitions — only the last iteration's delta is returned.
- Do **not** assume namespace safety for the node itself — if `agent-x:claim-42` did not previously exist with an `agent-x:active` dependency (e.g., added by `add_node(namespace=...)`), the trust-toggling invariant may be missing. Prefer using `add_node(namespace=...)` for new namespaced nodes.

---

## Dependencies

- **`reasons.Justification`** — the justification value object (`type`, `antecedents`, `outlist`, `label`, `content_hash`).
- **`reasons.api._with_network`** — the load/mutate/save/pubsub context manager. Central to correctness: writes only persist on clean exit.
- **`reasons.api._resolve_namespace`** — prefixing logic; skips IDs already containing `":"`.
- **`reasons.api._pg_dispatch`** — lazy-imports `PgApi` for the Postgres path.
- **`reasons.Network.add_justification`** (via `net.add_justification`) — actual recompute, truth-value flip detection, and dependents-index maintenance.
- **`reasons.pubsub`** — invoked indirectly by `_with_network.__exit__` when subscribers exist.

---

## Topics to Explore

- [function] `reasons/network.py:Network.add_justification` — The underlying implementation that determines the exact return-dict shape (`old_truth_value`, `new_truth_value`, `changed`) and how many times propagation runs during `any_mode`.
- [function] `reasons/api.py:add_node` — Contrast the namespace handling: `add_node` injects `{namespace}:active` into justifications and auto-creates the namespace premise, `add_justification` does neither.
- [file] `tests/test_add_justification.py` — Reveals the expected behavior for `any_mode`, `unless`-only, overlapping `sl`+`cp`, and namespace resolution.
- [file] `tests/test_dangling_dependents.py` — Explores what happens when antecedents or outlist IDs do not exist — important because `add_justification` does not pre-validate them.
- [general] `any-mode-OR-vs-AND-semantics` — How OR expansion is modeled (multiple single-antecedent justifications rather than a special `ANY` type), and why `premise_count` is hardcoded to `1` in that branch.

---

## Beliefs

- `sl-takes-precedence-over-cp` — When both `sl` and `cp` are non-empty, `cp` is silently ignored due to the `elif cp:` branch.
- `any-mode-is-sl-only-and-single-antecedent-noop` — `any_mode` only expands when `jtype == "SL"` and `len(antecedents) > 1`; it is otherwise silently ignored (including for CP).
- `any-mode-reports-premise-count-of-one` — In the `any_mode` branch the returned `premise_count` is hardcoded to `1` regardless of how many single-antecedent justifications were created.
- `namespace-resolution-does-not-add-active-antecedent` — Unlike `add_node`, `add_justification` resolves namespace prefixes on antecedents/outlist but never injects `{namespace}:active` as an antecedent.
- `unless-only-produces-sl-with-empty-antecedents` — Providing only `unless` (no `sl`/`cp`) creates an SL justification with `antecedents=[]`, matching the "premise that holds unless something is believed" pattern.
- `write-persistence-requires-clean-context-exit` — All mutations are only persisted (and pub/sub events published) if the `_with_network(write=True)` block exits without raising.
- `postgres-path-rejects-any-mode` — `pg_conninfo` combined with `any_mode=True` raises `NotImplementedError` before any network object is opened.
