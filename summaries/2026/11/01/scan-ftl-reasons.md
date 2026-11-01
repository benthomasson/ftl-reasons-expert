---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Scan: ftl-reasons

# Codebase Reconnaissance: ftl-reasons

## 1. Critical Files

| # | File | Why It Matters |
|---|------|---------------|
| 1 | `reasons/__init__.py` | Public API surface — defines what the package exports, reveals the intended interface and internal vs. external boundary |
| 2 | `reasons/network.py` | **The core engine.** Implements the TMS dependency graph: node states, justification evaluation, retraction cascades, restoration. This is where Doyle's algorithm lives |
| 3 | `reasons/storage.py` | Data layer (SQLite). Defines the schema, serialization of nodes/justifications/nogoods. Understanding persistence shapes understanding of the domain model |
| 4 | `reasons/derive.py` | Justification evaluation logic — SL (support list), CP (consistency premise) semantics. The "why is this node IN/OUT?" computation |
| 5 | `reasons/repair.py` | Dependency-directed backtracking. When a nogood fires, this traces back through the justification graph to find the minimal premise to retract. The most algorithmically complex part |
| 6 | `reasons/contradictions.py` | Nogood detection and classification. Bridges the gap between "two nodes are IN" and "here's the contradiction to resolve" |
| 7 | `reasons/cli.py` | Primary entry point. Shows how all components compose into user-facing operations. The "readme in code" for the system |
| 8 | `reasons/verify.py` | Verification/consistency checking. Reveals the invariant conditions the TMS must maintain |
| 9 | `reasons/compact.py` | Token-budgeted summarization. Shows how the network is projected into LLM-consumable context — reveals the "read model" of the system |
| 10 | `reasons/api.py` | Programmatic API layer. Shows the intended integration surface for external systems (agents, other tools) |
| 11 | `reasons/mcp_server.py` | MCP (Model Context Protocol) server. The secondary entry point — reveals how AI agents interact with the TMS |
| 12 | `reasons/llm.py` | LLM abstraction. Understands how optional LLM features (summarization, derivation assistance) plug in |
| 13 | `reasons_lib/network.py` | The "library" variant. Comparing against `reasons/network.py` reveals what's stripped for the lighter distribution |

## 2. Architecture Sketch

This is a **single-domain formal system** implementing Doyle's 1979 Truth Maintenance System. The architecture is a **flat service-layer design**: a core TMS engine (`network.py` + `derive.py` + `repair.py`) sits atop a SQLite persistence layer (`storage.py`), with thin interface adapters (CLI, MCP server, programmatic API) wrapping the same domain operations. LLM integration is a pluggable enhancement for summarization and derivation assistance, not part of the core inference. The `reasons` / `reasons_lib` duplication suggests a "full" vs. "library" distribution split, with `reasons` adding merkle proofs and metadata on top. There is no microservice boundary or event bus in the core — propagation is synchronous within a single process, with `pubsub.py` and `mind_service.py` providing optional external notification hooks.

## 3. Module Map

```
ftl-reasons/
├── reasons/              ← FULL package (with merkle, metadata)
│   ├── Core TMS engine   → network.py, derive.py, repair.py, repair_premises.py
│   ├── Domain logic      → contradictions.py, verify.py, cluster.py, compact.py
│   ├── I/O & interfaces  → cli.py, api.py, mcp_server.py, mcp_client.py
│   ├── Persistence       → storage.py, pg.py (PostgreSQL variant), merkle.py
│   ├── LLM integration   → llm.py, hf.py (HuggingFace), mind_service.py
│   ├── Import/Export     → import_beliefs.py, import_agent.py, export_card.py,
│   │                       export_markdown.py, build_wiki.py
│   └── Operations        → propose_update.py, report_belief.py, report_gated.py,
│                           review*.py, check_stale.py, pubsub.py, metadata.py
│
├── reasons_lib/          ← LIBRARY package (lighter, no merkle)
│   └── (same modules minus merkle.py — shared domain logic)
│
├── tests/                ← 211+ tests, well-organized by feature
├── .github/workflows/    ← CI: pytest
└── .claude/skills/       ← Claude skill definition for the "reasons" tool
```

**Responsibility clusters:**
- **TMS Core** (invariant logic): `network`, `derive`, `repair`, `repair_premises`, `contradictions`
- **Persistence & Integrity**: `storage`, `pg`, `merkle`, `metadata`
- **User Interfaces**: `cli`, `api`, `mcp_server`, `mcp_client`
- **LLM / AI Integration**: `llm`, `hf`, `mind_service`, `compact`, `cluster`
- **Import/Export Ecosystem**: `import_*`, `export_*`, `build_wiki`
- **Governance / Review**: `review*`, `report_*`, `propose_update`
- **Lifecycle**: `check_stale`, `verify`, `pubsub`

## 4. Exploration Strategy

Start from the **outside in** (interface → engine → storage), because the README and CLI already tell you *what* the system does; the code tells you *how*. Then dive into the algorithmic core.

```
Phase 1: Interface & Contract
  1. __init__.py        → public API, what's exported
  2. cli.py             → command surface, how components compose
  3. api.py             → programmatic contract

Phase 2: Core Engine (the TMS)
  4. network.py         → node/justification model, propagation loop
  5. derive.py          → SL/CP evaluation semantics
  6. repair.py          → dependency-directed backtracking
  7. contradictions.py  → nogood detection & classification

Phase 3: Persistence & Invariants
  8. storage.py         → schema, serialization
  9. verify.py          → consistency invariants
 10. merkle.py          → (reasons only) integrity proofs

Phase 4: LLM & Extensions
 11. compact.py         → how the network is projected for LLMs
 12. llm.py             → LLM abstraction layer
 13. reasons_lib/network.py → diff vs. reasons/network.py
```

---

## Topics to Explore

- [function] `reasons/__init__.py` — Public API surface: reveals what the package exposes, how `reasons` and `reasons_lib` differ, and the intended integration contract
- [function] `reasons/cli.py:main` — Primary entry point: shows how all domain operations compose into user-facing commands (add, retract, assert, nogood, challenge, defend, propagate, etc.)
- [general] `tms-node-model` — The core data model: nodes (IN/OUT), justifications (SL, CP, outlist), nogoods, and their relationships. Understood via `network.py`
- [function] `reasons/network.py:propagate` (or equivalent) — The propagation engine: how retraction cascades and restoration work. The heart of Doyle's algorithm
- [function] `reasons/derive.py` — Justification evaluation: how SL (all antecedents IN) and CP (consistency) justifications are checked, including outlist (non-monotonic) semantics
- [function] `reasons/repair.py` — Dependency-directed backtracking: how the system traces a nogood backward through the justification graph to identify the minimal responsible premise to retract
- [function] `reasons/contradictions.py` — Nogood detection and defeater classification: the bridge between "contradiction exists" and "here's what to retract"
- [function] `reasons/storage.py` — SQLite schema and persistence: the canonical record of the domain model (tables, serialization format, round-trip guarantees)
- [function] `reasons/verify.py` — Consistency invariants: what the system guarantees about the network state after any operation. Reveals the "postconditions" of the TMS
- [function] `reasons/compact.py` — Token-budgeted summarization: how the dependency network is projected into LLM-consumable text. Reveals the "read model" and importance ranking
- [function] `reasons/api.py` — Programmatic API: the integration surface for external agents and tools. Shows which operations are first-class vs. internal
- [function] `reasons/mcp_server.py:main` — MCP server entry point: how AI agents (Claude, etc.) interact with the TMS. Reveals the tool-call contract
- [function] `reasons/llm.py` — LLM abstraction layer: how optional LLM capabilities (summarization, derivation assistance) are plugged in without coupling the core TMS
- [general] `reasons-vs-reasons_lib` — The distribution split: comparing `reasons/merkle.py` and `reasons/metadata.py` (present) vs. `reasons_lib/` (absent or different) reveals what's "full" vs. "library"
- [general] `dialectical-argumentation` — The challenge/defend mechanism: how non-monotonic outlists enable formal belief challenging and defense. Spans `network.py`, `repair.py`, `contradictions.py`
