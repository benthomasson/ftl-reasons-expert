---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: README.md

# README.md — Project Documentation for `ftl-reasons`

## 1. Purpose

This is the **project-level documentation** — the single source of truth for what `ftl-reasons` is, how to install it, how to use its CLI, and what theoretical underpinning it implements. It owns three responsibilities:

- **Conceptual onboarding**: It translates Doyle's (1979) Truth Maintenance System into working vocabulary (nodes, SL/CP justifications, nogoods, outlists) so a reader can reason about the system before touching code.
- **Operational contract**: It defines every CLI command, its flags, and expected output format. This is effectively the API contract for the `reasons` binary.
- **Integration surface**: It documents import/export formats (`beliefs.md`, `nogoods.md`, JSON), which are the boundaries between this system and external tools (agent registries, LLM context injection via `compact`).

It does **not** document the internal architecture, the `reasons/` vs. `reasons_lib/` split, the MCP server, the HuggingFace integration, the PostgreSQL layer, or the merkle tree — none of those appear here. Those live in `CLAUDE.md` or the code itself.

## 2. Key Components (as documented)

The README defines the **domain model** through its "Core Ideas" section:

| Concept | Contract |
|---------|----------|
| **Node** | Has a truth value: `IN` (believed) or `OUT` (retracted). Is either a *premise* (independent truth) or *derived* (truth depends on justifications). |
| **SL justification (Support List)** | Node is IN ⟺ **all** antecedents in the inlist are IN **and** all nodes in the outlist are OUT. Multiple justifications: node is IN if **any** justification is valid. |
| **CP justification** | Assumptions must be *consistent* (documented in the command table but not elaborated in Core Ideas). |
| **Outlist** | Non-monotonic guard: "believe X unless Y." A justification is valid only when all outlist nodes are OUT. |
| **Nogood** | A set of nodes that cannot all be IN. Triggers **dependency-directed backtracking** to find the responsible premise. |
| **Retraction cascade** | When a node goes OUT, all dependents whose justifications become invalid recursively go OUT. |
| **Restoration** | When a retracted node returns IN, dependents are automatically recomputed — no rederivation needed. |
| **Challenge / Defend** | Dialectical argumentation: `challenge` forces a target OUT; `defend` restores it and retracts the challenge. |

The **command table** (the 25-row table) is the functional API. Every row is a contract: `reasons add ID "text" --sl a,b` means "create a derived node whose sole justification requires `a` and `b` to both be IN."

## 3. Patterns

- **Theoretical grounding → practical mapping**: Each concept in "Core Ideas" maps 1:1 to a CLI verb. `SL justification` → `--sl` flag. `Nogood` → `reasons nogood`. `Outlist` → `--unless`. This is a deliberate pedagogical pattern: read theory, then see the command.
- **Cascade as first-class semantics**: The README repeatedly emphasizes that operations are *not* atomic. `retract source-uses-langgraph` doesn't just flip one node — it triggers a cascade. The example output (`Retracted: source-uses-langgraph, topology-is-static, no-runtime-modification`) makes this contract explicit.
- **Round-trip symmetry**: `import-beliefs` ↔ `export-markdown` are documented as inverse operations on the `beliefs.md` format. `export` (JSON) is the lossless form.
- **Two installation paths, one binary**: `pip install` and `uv tool install` both produce the `reasons` CLI entry point. `uvx` is the zero-install path. All three converge on the same command surface.
- **Duplicate module directories**: The repo tree shows `reasons/` and `reasons_lib/` with nearly identical file sets (the only difference: `reasons/` has `merkle.py`, `reasons_lib/` doesn't; `reasons/` also has `pubsub.py` and `network.py` that `reasons_lib/` also has). The README doesn't explain this split — it's a maintenance concern, not a user-facing one.

## 4. Dependencies

**What this file depends on (conceptually):**
- The `reasons` CLI binary and its subcommands (documented in the command table).
- The `beliefs.md` / `nogoods.md` file format (import/export contract).
- Doyle's (1979) paper as the theoretical reference.
- `pyproject.toml` for the package name `ftl-reasons` and the test extra (`--extra test`).

**What depends on this file:**
- New contributors (onboarding entry point).
- CI/CD workflows in `.github/workflows/` (the test command `uv run --extra test pytest tests/ -v` is specified here).
- `CLAUDE.md` (which likely references or supplements this for agent-specific instructions).
- Downstream projects that integrate via `import-beliefs` or the MCP server (`mcp_server.py`) — the README defines the interchange format.

**What this file does NOT depend on:**
- `reasons/` vs. `reasons_lib/` internal structure.
- `merkle.py`, `pg.py`, `hf.py`, `mcp_client.py`, `mcp_server.py`, `mind_service.py` — none are mentioned. These are internal or optional integrations.

## 5. Flow

The README doesn't describe code execution flow — it's documentation. But it implicitly defines the **runtime flow** a user follows:

```
init → add (premises + derived) → status
  → retract (cascade OUT) → status
  → assert (cascade IN / restoration) → status
  → nogood (backtrack to premise)
  → challenge / defend (dialectical)
  → explain / trace / show (inspection)
  → export / export-markdown (persist)
  → import-beliefs (restore)
  → check-stale → hash-sources (integrity)
  → compact (LLM context)
```

The data transformation is: **beliefs.md (markdown) → in-memory TMS graph → SQLite (reasons.db) → JSON/markdown export**. The `reasons.db` is the persistent store (referenced by `reasons init` creating `reasons.db` and `reasons export` reading from it).

## 6. Invariants

The README encodes several invariants that the implementation must uphold:

1. **Any-justification rule**: A node is IN if *any* of its justifications is valid. This is stated explicitly: "A node can have multiple justifications — it's IN if *any* of them are valid."
2. **Outlist universality**: For a justification to be valid, *all* outlist nodes must be OUT. There's no partial satisfaction.
3. **Cascade completeness**: Retraction is transitive. The example shows `source-uses-langgraph` → `topology-is-static` → `no-runtime-modification` all going OUT. The implementation must not stop at one level.
4. **Restoration symmetry**: Asserting a node back IN must restore all dependents that were retracted *because of* that node. The example confirms: `assert source-uses-langgraph` brings back both `topology-is-static` and `no-runtime-modification`.
5. **Nogood irreversibility (within a set)**: Once a nogood is recorded, the set of nodes "cannot all be IN simultaneously." Backtracking finds a *premise* to retract, not an arbitrary node.
6. **Premise vs. derived distinction**: Premises have no justifications; they are IN by default. Derived nodes have ≥1 justification. This is structural, not a flag.
7. **Source hash stability**: `check-stale` compares stored hashes against current file state. `hash-sources` backfills missing hashes. The invariant is: a node with a `--source` flag should eventually have a hash, and a mismatch means the source has changed.

## 7. Error Handling

The README is silent on error handling — no error messages, exit codes, or failure modes are documented. This is a **gap**:

- What happens if you `retract` a node that's already OUT?
- What happens if `--sl` references a non-existent node ID?
- What does `import-beliefs` do with a malformed `beliefs.md`?
- What's the exit code on cascade failure vs. success?
- How does `check-stale` report a deleted source file vs. a modified one?

The test suite (211 tests) likely covers these, but the README doesn't document user-facing error semantics. A maintainer should cross-reference `tests/` (especially `test_check_stale_issue25.py`, `test_dangling_dependents.py`, `test_deduplication.py`) to understand the expected error behavior.

---

## Topics to Explore

- [file] `reasons/network.py` — The core TMS engine: implements node truth propagation, SL/CP justification evaluation, retraction cascades, and restoration. Understanding this file makes the README's "Core Ideas" section concrete.
- [file] `reasons/contradictions.py` — Nogood detection and dependency-directed backtracking. The README describes the *what* (retract the responsible premise with minimal disruption) but not the *how* (graph traversal order, minimality criterion).
- [file] `reasons/repair.py` and `reasons/repair_premises.py` — The README mentions "repair" in the repo tree but never in the command table or usage examples. These likely handle nogood resolution strategies not exposed in the current CLI.
- [diff] `reasons/` vs. `reasons_lib/` — Two nearly identical package directories. `reasons/` has `merkle.py` and `pubsub.py`; `reasons_lib/` does not. Understanding which is the "real" package and which is a vendored/stripped copy is essential before modifying any shared module.
- [general] `mcp-server-integration` — `mcp_server.py` and `mcp_client.py` exist in both package dirs but are completely absent from the README. If this system is meant to be queried by LLM agents via MCP, that's a major interface the documentation doesn't cover.

## Beliefs

- `node-in-requires-any-justification-valid` — A derived node is IN if and only if at least one of its justifications is valid; the README explicitly states "it's IN if *any* of them are valid," making this a disjunction over justifications, not a conjunction.
- `retract-cascade-is-transitive` — Retracting a node propagates OUT status through the entire dependent subgraph, not just one hop; the example shows a 3-level cascade (`source-uses-langgraph` → `topology-is-static` → `no-runtime-modification`).
- `beliefs-md-is-roundtrip-stable` — `import-beliefs` and `export-markdown` are inverse operations on the `beliefs.md` format; a node graph exported then re-imported must produce an identical TMS state (validated by `test_roundtrip.py`).
- `nogood-backtracking-targets-premises-only` — Dependency-directed backtracking after a nogood is detected retracts a *premise* node, not a derived node; the README states "retract the responsible *premise* with minimal disruption — not an arbitrary node."
- `outlist-enables-default-reasoning` — The `--unless` flag (outlist) makes a justification conditionally valid: the node is IN only when all outlist nodes are OUT, which is the formal mechanism for "believe X unless Y" default reasoning.
