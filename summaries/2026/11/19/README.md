---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: README.md

# README.md — Explanation

## 1. Purpose

`README.md` is the **public contract document** for the `reasons` project. It is not just marketing copy — it is the only place in the repo where the CLI surface is specified. Concretely it owns four responsibilities:

1. **Domain model definition** — defines the vocabulary (node, IN/OUT, premise, derived, justification, SL/CP, outlist, nogood, retraction cascade, dependency-directed backtracking) that every module in `reasons/` is written against.
2. **Install/distribution spec** — names the PyPI distribution (`ftl-reasons`) and the three install paths (`pip`, `uv tool install`, `uvx`), which must stay in sync with `pyproject.toml`.
3. **CLI reference** — the `Usage` narrative plus the `Commands` table (≈22 verbs) is the authoritative list of user-facing commands and their flags.
4. **Test invocation contract** — the `Tests` section specifies `uv run --extra test pytest tests/ -v` and asserts a headline count ("211 tests").

It sits beside `CHANGELOG.md` and `CLAUDE.md` at repo root; `CLAUDE.md` is the agent-facing counterpart, `CHANGELOG.md` the temporal counterpart.

A key observation for a maintainer: **the README documents the CLI, never the Python API.** No module path, class name, or function signature appears anywhere in the file. That is deliberate (the library is consumed via the `reasons` executable), but it means the README cannot be used to navigate `reasons/*.py` — the verb→module mapping is inferred, not stated.

---

## 2. Key Components

Since this is prose, the "components" are the concepts and command contracts it defines.

### 2.1 Domain entities

| Entity | Definition in README | Contract |
|---|---|---|
| **Node** | A belief with truth value `IN` (believed) / `OUT` (retracted) | Uniquely identified by an ID string (kebab-case by convention, e.g. `source-uses-langgraph`); carries a human-readable text |
| **Premise** | A node believed by default | IN unless explicitly retracted or defeated by a nogood |
| **Derived node** | A node believed because of justifications | IN iff ≥ 1 justification is valid; never independently asserted |
| **Justification (SL)** | "Support List" — valid iff *all* inlist antecedents are IN | Monotonic; multiple SL justifications on one node = OR semantics |
| **Justification (CP)** | `--cp a,b` — assumptions must be *consistent* | Consistent-conditional-proof; validity includes a global consistency check against nogoods |
| **Outlist** | `--unless y` — believe X unless Y is believed | Non-monotonic; valid iff all inlist IN **and** all outlist nodes OUT |
| **Nogood** | A set of nodes that cannot all be IN simultaneously | Auto-IDed (`nogood-001`); at least one member must end OUT |
| **Source** | `--source "agents-python:src/graph.py"` | `name:path` provenance; hashed for staleness detection |
| **Label** | `--label "observed from source"` | Free text attached to a justification, surfaced in `explain` in brackets |

### 2.2 Command contracts (from the table)

**Mutation (write to `reasons.db`)**
- `reasons init` — creates the SQLite database. Must be idempotent or fail cleanly; every other command implicitly requires it.
- `reasons add ID "text"` — premise; variants `--sl a,b`, `--sl a --unless y`, `--cp a,b`, `--source`, `--label`.
- `reasons retract ID` — sets OUT **and** prints the full cascade set.
- `reasons assert ID` — sets IN **and** prints the full restoration set.
- `reasons nogood A B ...` — records a contradiction then backtracks; prints both the nogood ID and the retraction set.
- `reasons challenge ID "reason"` / `reasons defend TARGET CHALLENGE "reason"` — create synthetic challenge/defense nodes that attack/support the target.
- `reasons propagate` — full recompute (repair/drift recovery).
- `reasons hash-sources [--force]` — backfill source hashes; `--force` re-hashes already-hashed nodes.

**Read (no state change)**
- `reasons status` — sorted `[+]`/`[-]` listing plus an `N/M IN` footer.
- `reasons show ID`, `reasons explain ID`, `reasons trace ID`, `reasons log`, `reasons search QUERY`, `reasons list` (with `--status`, `--premises`, `--has-dependents`, `--challenged`).
- `reasons check-stale` — hash comparison for **IN nodes only**.
- `reasons compact --budget N [--no-truncate]` — token-budgeted context injection.

**Interchange**
- `reasons export` → JSON; `reasons export-markdown [-o FILE]` → beliefs.md format.
- `reasons import-beliefs FILE [--nogoods PATH]` — auto-detects `nogoods.md` as a sibling of `beliefs.md`.

### 2.3 Implied module mapping

The repo layout makes the verb→module correspondence nearly 1:1, and a maintainer should treat this as the operative index:

`init/add/retract/assert/status/show/explain/log/search/list` → `reasons/cli.py` (+ `storage.py`, `derive.py`, `verify.py`); `check-stale` → `reasons/check_stale.py`; `compact` → `reasons/compact.py`; `export-markdown` → `reasons/export_markdown.py`; `import-beliefs` → `reasons/import_beliefs.py`; `challenge/defend`/nogood handling → `reasons/contradictions.py` and `reasons/verify.py`.

**Gap to flag:** the module tree contains ~13 modules with *no* README coverage — `mcp_server.py`, `mcp_client.py`, `llm.py`, `cluster.py`, `review*.py`, `repair*.py`, `propose_update.py`, `report_*.py`, `mind_service.py`, `network.py`, `pg.py`, `pubsub.py`, `hf.py`, `merkle.py`, `metadata.py`, `build_wiki.py`, `export_card.py`, `import_agent.py`, `ask.py`, `derive.py`. Either they are unreleased or the README is stale.

---

## 3. Patterns

- **Documentation-as-contract with inline golden output.** Every `Usage` example interleaves the command with commented expected stdout (`# 4/4 IN`, `# Retracted: source-uses-langgraph, topology-is-static, ...`). These are effectively assertion fixtures and should mirror `tests/test_cli.py`.
- **Doyle TMS vocabulary preserved verbatim** — *inlist*, *outlist*, *nogood*, *dependency-directed backtracking*, *premise/derived* come straight from Doyle (1979), which is cited explicitly. Do not rename these concepts in code without updating the reference.
- **Dependency-directed backtracking (DDB) as the contradiction-resolution strategy.** The README is explicit that the system "trace[s] backward through the justification graph and retract[s] the responsible *premise* with minimal disruption — not an arbitrary node." That is the distinguishing algorithmic claim of the project.
- **Argumentation layered on TMS primitives.** `challenge`/`defend` are not a separate engine; they compose on top of the same justification graph (`challenge-velocity-constraint` and `defense-challenge-velocity-constraint` look like ordinary nodes with SL/outlist edges).
- **Round-trip interchange symmetry.** `export-markdown` emits "beliefs.md-compatible markdown" and `import-beliefs` consumes exactly that shape — a symmetry likely exercised by `tests/test_roundtrip.py`.
- **Provenance-by-hash.** Sources are tracked as `name:path` + content hash; `check-stale` and `hash-sources` are the two-phase backfill/verify idiom (`--force` = idempotency escape hatch).
- **One module per CLI verb** in `reasons/`, with `reasons_lib/` as a mirror tree — the classic "shim package" layout for a renamed project.
- **Token-budgeted output** (`compact --budget N`) as a first-class command — signals the tool is designed to be piped into an LLM context window, which explains the presence of `llm.py`, `ask.py`, and `mcp_server.py`.

---

## 4. Dependencies

**What `README.md` depends on (things that can make it wrong):**

| Dependency | Why it matters |
|---|---|
| `pyproject.toml` | Must define distribution `ftl-reasons` and a `reasons` console-script entry point, plus a `test` extra |
| `reasons/cli.py` | Must register every verb in the `Commands` table with matching flag names |
| `tests/` | The claimed "211 tests" and the enumerated test coverage list must match reality |
| `CHANGELOG.md` | Any behavioral change in the table must be announced there; README describes *current* behavior only |

**What depends on it:**

- `pyproject.toml` — the `readme` field is almost certainly `README.md`, making it the PyPI landing page. Markdown syntax errors or broken fenced blocks are user-visible on the package index.
- Human onboarding, and `CLAUDE.md`, which typically imports or mirrors the CLI contract.
- Any downstream consumer writing scripts against `reasons <verb>`.

**Notably absent:** there are no imports or code references. The README never names `reasons.storage`, `reasons.verify`, etc.

---

## 5. Flow

The README's `Usage` section is a carefully ordered **narrative that exercises the engine in increasing complexity**. Read it as a state-machine walkthrough:

1. **`init`** — creates `reasons.db` (SQLite, per `reasons/pg.py`/`storage.py` split; local default is SQLite, `pg.py` suggests Postgres is optional).
2. **Assert premises** — `add source-uses-langgraph ... --source "agents-python:src/graph.py"`, `add graph-has-cycles ...`. Both are `[+]`.
3. **Build a derivation chain** — `topology-is-static --sl source-uses-langgraph --label "observed from source"`, then `no-runtime-modification --sl topology-is-static`. `status` shows `4/4 IN`. Note the deferred-evaluation contract: derived nodes are never manually asserted, only recomputed.
4. **Retract → cascade** — `retract source-uses-langgraph` prints `Retracted: source-uses-langgraph, topology-is-static, no-runtime-modification`. Independently-grounded `graph-has-cycles` survives: `1/4 IN`. The output order implies a topological walk from the retracted node downward.
5. **Assert → restoration** — `assert source-uses-langgraph` prints `Asserted: source-uses-langgraph, topology-is-static, no-runtime-modification`. Exactly the same set, restored **without rederivation** — restoration re-evaluates justifications rather than replaying the derivation.
6. **Contradiction → DDB** — `add graph-is-dynamic`, then `nogood topology-is-static graph-is-dynamic` prints `Recorded nogood-001: ...` and `Retracted: graph-is-dynamic`. Both members were premises; the engine picked `graph-is-dynamic`, presumably because `topology-is-static` has more dependents (minimal-disruption heuristic).
7. **Explain** — walks the justification tree downward and renders provenance labels: `[+] topology-is-static: SL justification valid — antecedents: source-uses-langgraph [observed from source]`, bottoming out at `[+] source-uses-langgraph: premise`. This is a read-only projection of the graph.
8. **Non-monotonic round-trip** — `add default-approx ... --unless strong-field`; `assert strong-field` drives `default-approx` OUT; `retract strong-field` restores it. Symmetric with step 4/5 but mediated by an outlist rather than an inlist.
9. **Transfer in/out** — `import-beliefs beliefs.md` → `import-beliefs`; then cascade works on the imported graph (`retract beliefs-improve-accuracy` cascades through imported dependents). `export-markdown` closes the loop.
10. **Argumentation** — `challenge` retracts the target and cascades; `defend` retracts the challenge node, which restores the target and its dependents. This is the outlist mechanism reused: the defense attacks the *challenge node*, not the original target.

**Data transformation summary:** CLI argv → parsed verb+flags → SQLite mutation → propagation recompute → stdout rendering (`[+]`/`[-]` + `N/M IN` footer). The `log` command exposes the propagation audit trail as an append-only record of step 3–8 transitions.

---

## 6. Invariants

These are the rules a maintainer must not break, each traceable to a README sentence:

1. **Truth-value invariant:** a node is IN iff (a) it is a premise that has not been retracted, or (b) at least one of its justifications is valid. There is no third source of IN.
2. **SL validity:** valid iff **all** inlist antecedents are IN. One OUT antecedent invalidates the justification, but not the node if another justification holds.
3. **Outlist validity:** valid iff all inlist nodes are IN **and** all outlist nodes are OUT. Adding an outlist node to an already-valid SL justification makes it strictly harder to satisfy.
4. **Nogood invariant:** for every recorded nogood `{n1..nk}`, at least one member must be OUT at all times. Violating this is what `propagate` repairs.
5. **Cascade transitivity:** retracting node X must put OUT every node reachable via justification invalidation, and the printed set must be closed under that relation.
6. **Restoration symmetry:** asserting X restores exactly the nodes whose justifications became valid — the README asserts this happens "without rederivation," i.e. it is a pure function of the current graph, not a replay of history.
7. **Premise independence:** retracting a premise that has no dependents must not affect sibling premises (`graph-has-cycles` stays `[+]` at `1/4 IN`).
8. **DDB targets premises, not arbitrary nodes:** contradiction resolution retracts a *premise* reachable backward through the justification graph, chosen for minimal disruption.
9. **ID uniqueness:** IDs are the primary key and are used as CLI handles; text is descriptive only and may be non-unique.
10. **`check-stale` scopes to IN nodes:** the README says "Check **IN nodes** for source file hash changes" — OUT nodes are not hash-checked. `hash-sources` backfills all unhashed nodes, `--force` re-hashes regardless.
11. **`--budget N` is a hard token ceiling for `compact`**, with `--no-truncate` as the opt-out.

---

## 7. Error Handling

**The README specifies almost no error semantics — this is the largest documentation gap.** What it does and does not imply:

- **Nothing on exit codes.** No mention of non-zero status for "unknown node ID", "justification references nonexistent antecedent", "nogood that cannot be resolved", or "database not initialized". A developer building shell automation on top of `reasons` cannot tell from this file whether failures are signalled by exit code, stderr, or an exception traceback.
- **Nothing on unresolvable contradictions.** Step 6 shows a nogood where both members are premises and one *is* retracted. The README does not say what happens when a nogood's members are all non-retractable (e.g., mutually-supporting derived nodes with no retractable premise) — whether `nogood` errors, leaves the constraint unsatisfied, or cascades to a deterministic fallback.
- **The "success output" transcripts function as implicit contracts** — `Retracted: ...`, `Asserted: ...`, `Recorded nogood-001: ...`, `Defended ... with ...`, `Changed: ...` — so a maintainer should assume these exact prefixes are asserted in `tests/test_cli.py` and will break if reworded.
- **`propagate` is the escape hatch** for state that has drifted inconsistent (e.g., after a partial `add` or manual SQL). Its existence implies the normal commands do *not* guarantee a full recompute on every mutation, only incremental cascade.
- **`hash-sources --force`** is the idempotency escape hatch for stale/corrupt hash metadata.
- **`import-beliefs` degrades gracefully on partial input**: it auto-detects `nogoods.md` and prints `Imported 39 claims (2 retracted)` / `Imported 8 nogoods`, implying retracted state survives import rather than being renormalized.

If you extend this file, error semantics (exit codes, unknown-ID behavior, unresolvable-nogood behavior) are the highest-value additions.

---

## Topics to Explore

- [file] `reasons/cli.py` — The authoritative registration point for every verb in the `Commands` table; check whether the undocumented modules (`mcp_server`, `cluster`, `review*`, `repair*`) also register commands, which would prove the README is stale.
- [file] `reasons/verify.py` — Almost certainly hosts the propagation/justification-validity evaluator that enforces invariants 1–3 and the retraction/restoration cascade in invariants 5–7.
- [file] `reasons/contradictions.py` — Implements nogood detection and the minimal-disruption premise selection behind step 6 (`Recorded nogood-001` → `Retracted: graph-is-dynamic`); the heuristic that picks *which* premise to blame lives here.
- [diff] `reasons/ vs reasons_lib/` — The two package trees are file-for-file identical except `reasons/merkle.py` has no counterpart in `reasons_lib/`. Determine whether `reasons_lib` is a compatibility shim or has silently drifted.
- [general] `readme-code-drift-audit` — Cross-check each `Usage` transcript against `tests/test_cli.py` golden output, verify the "211 tests" claim, and flag the ~13 modules with zero README coverage.

## Beliefs

- `readme-documents-cli-surface-not-python-api` — `README.md` specifies ~22 CLI verbs and their flags but never names a Python module, class, or function, so it cannot be used to navigate `reasons/*.py`.
- `distribution-name-differs-from-executable-name` — The installable distribution is `ftl-reasons` (`pip install ftl-reasons`, `uv tool install ftl-reasons`) while the console executable is `reasons`.
- `node-is-in-iff-premise-or-valid-justification` — A node is IN if and only if it is an unretracted premise or at least one of its justifications is valid under SL (`all inlist IN`), CP (assumptions consistent), or outlist (`inlist IN and outlist OUT`) semantics.
- `retraction-cascade-enables-rederivation-free-restoration` — Retracting a node transitively marks invalidated dependents OUT, and re-asserting it restores exactly that set by re-evaluating justifications rather than replaying derivations.
- `nogood-resolution-retracts-a-premise-not-an-arbitrary-node` — Recording a nogood triggers dependency-directed backtracking that walks the justification graph backward and retracts a responsible premise chosen for minimal disruption.
- `export-markdown-roundtrips-with-import-beliefs` — `reasons export-markdown` emits beliefs.md-compatible markdown that `reasons import-beliefs` consumes, with `nogoods.md` auto-detected as a sibling of `beliefs.md`.
- `check-stale-scopes-hash-comparison-to-in-nodes` — Staleness detection compares stored source hashes only for IN nodes; `hash-sources` backfills unhashed nodes and `--force` re-hashes all.
