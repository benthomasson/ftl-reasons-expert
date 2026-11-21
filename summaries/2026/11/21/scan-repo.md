---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Scan: repo

# Repository Reconnaissance: `ftl-reasons` (Truth Maintenance System)

## Critical Files

| # | File | Why it matters |
|---|------|----------------|
| 1 | `pyproject.toml` | Declares **two packages** (`reasons`, `reasons_lib`), the real entry points (`reasons.cli:main`, `reasons.mcp_server:main`), and the optional extras that reveal hidden subsystems (`pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`, `hf`). Zero runtime deps in the core — that's a strong signal the TMS engine is self-contained. |
| 2 | `README.md` | The **semantics contract**: IN/OUT, SL justifications, outlist (non-monotonic), nogoods, dependency-directed backtracking, challenge/defend. Every core module should be understandable through this lens. |
| 3 | `CLAUDE.md` | Repo-specific conventions and likely architectural notes the author wants agents to follow — read before editing. |
| 4 | `reasons/cli.py` | Primary entry point. The command table in the README maps roughly 1:1 to dispatch branches here — it's the fastest index into every subsystem. |
| 5 | `reasons/storage.py` | Data model + SQLite persistence: nodes, justifications, justificand antecedents, outlists, nogoods, dependents, propagation log. **Everything hinges on this schema.** |
| 6 | `reasons/verify.py` | Implements justification validity — the actual "is this SL/CP/outlist justification satisfiable?" predicate. This is the atomic unit of TMS reasoning. |
| 7 | `reasons/network.py` | The **propagation engine**: truth-value recomputation, retraction cascade, restoration. This is the heart of Doyle's algorithm. |
| 8 | `reasons/contradictions.py` | Nogood detection + **dependency-directed backtracking** — the hardest algorithm in the repo; traces back to the responsible premise with minimal disruption. |
| 9 | `reasons/derive.py` | Creates derived nodes with justifications; likely blended with LLM assistance. Connects TMS core to the higher-level "reasoning" workflows. |
| 10 | `reasons/api.py` | Programmatic surface used by CLI, MCP server, and tests — the shared type/shape boundary for the library. |
| 11 | `reasons/mcp_server.py` | Second official entry point; exposes the belief graph over MCP. Mirrors much of `cli.py` logic, so useful as a cross-check on API shape. |
| 12 | `reasons/merkle.py` | Present in `reasons/` but **missing from `reasons_lib/`** — likely the fastest tell for what the duplicated tree is (a snapshot? older vendored version? a fork?). |
| 13 | `reasons/llm.py` | Abstraction over Anthropic / Vertex / OpenAI / Langfuse. Used by `ask`, `review*`, `derive`, `propose_update`, `import_agent`. |
| 14 | `tests/conftest.py` | Fixtures + DB bootstrap; reveals how the schema is exercised and how commands are invoked in-process. |
| 15 | `tests/test_backtracking.py` | Behavioral spec of the hardest part of the system. Cheaper to read than the algorithm and authoritative about edge cases (diamonds, multiple justifications, restoration). |

**Deliberately not critical (though present):** `.forge/`, `costs/`, `.github/workflows/tests.yml`, `.claude/skills/reasons`, and everything under `reasons_lib/` *until* the duality with `reasons/` is resolved.

## Architecture Sketch

`ftl-reasons` is a **self-contained Truth Maintenance Engine** wrapped in multiple delivery surfaces. At the core sits a **justification/persistence layer** (`storage.py`) holding nodes, SL/CP/outlist justifications, nogoods, and a propagation audit log; above it, `verify.py` decides justification validity while `network.py` recomputes IN/OUT and drives retraction/restoration cascades, and `contradictions.py` performs dependency-directed backtracking when a nogood fires. Around this core sit **delivery surfaces** (`cli.py`, `api.py`, `mcp_server.py`) and **higher-level workflows** (`derive`, `ask`, `review`, `repair`, `propose_update`, `import_*`, `export_*`, `compact`, `build_wiki`) that use a pluggable LLM layer (`llm.py` + optional provider extras) and optional backends (`pg.py`, `hf.py`, `cluster.py`, `pubsub.py`/`mind_service.py`, `merkle.py`). The **duplicated `reasons/` + `reasons_lib/` trees** (with `merkle.py` and a few diffs) suggest either a vendored snapshot, a rename-so-far incompletely performed, or a "pure library" mirror — resolving this is the highest-leverage unknown in the repo.

## Module Map

- **Core TMS** — `storage.py`, `network.py`, `verify.py`, `contradictions.py`, `derive.py`, `merkle.py`. Pure logic + persistence.
- **Entry points / surfaces** — `cli.py`, `api.py`, `mcp_server.py`, `mcp_client.py`.
- **LLM-driven workflows** — `ask.py`, `review.py`, `review_premises.py`, `review_justifications.py`, `repair.py`, `repair_premises.py`, `propose_update.py`, `import_agent.py`, `report_belief.py`, `report_gated.py`.
- **Ingest / export** — `import_beliefs.py`, `import_agent.py`, `export_card.py`, `export_markdown.py`, `build_wiki.py`, `hf.py`.
- **Analysis / maintenance** — `check_stale.py`, `compact.py`, `cluster.py`, `contradictions.py`, `metadata.py`, `deduplication`-related tests.
- **Infrastructure** — `llm.py`, `pg.py`, `pubsub.py`, `mind_service.py`, `network.py` (tests hint at network caching; may be graph "network" or HTTP — verify).
- **Parallel package** — `reasons_lib/` (near-clone of `reasons/`, missing `merkle.py`; role TBD, likely the same intent as `reasons/`).
- **Tests** — `tests/` — one file per command/module; effectively the spec.
- **Non-code** — `.forge/` (tooling artifacts), `costs/` (LLM cost logs), `.claude/skills/reasons` (agent skill), `.github/workflows/tests.yml` (CI).

## Exploration Strategy

1. **Orientation** — `README.md` → `CLAUDE.md` → `pyproject.toml` → `.claude/skills/reasons`. Build the vocabulary (IN/OUT, SL, outlist, nogood, backtrack) before reading code.
2. **Resolve the `reasons` vs `reasons_lib` duality** — start with `diff reasons/cli.py reasons_lib/cli.py` and `ls reasons_lib/` (missing `merkle.py`). Do not explore `reasons_lib/` further until you know which tree is canonical.
3. **Entry points** — `reasons/cli.py` (command dispatch) → `reasons/api.py` (library surface) → `reasons/mcp_server.py` (second surface, cross-check).
4. **Core engine, schema first** — `storage.py` (schema) → `verify.py` (justification semantics) → `network.py` (propagation) → `contradictions.py` (backtracking) → `derive.py`. Read `tests/conftest.py` and `tests/test_backtracking.py` alongside to pin down invariants.
5. **Behavior confirmation** — `test_storage.py`, `test_outlist.py`, `test_dialectical.py`, `test_roundtrip.py` are the cheapest way to see the intended contracts in action.
6. **Integrations** — `llm.py`, `mcp_client.py`, `pg.py`, `merkle.py`, `hf.py`.
7. **Workflows outward** — `ask.py`, `review*.py`, `repair*.py`, `propose_update.py`, then `import_*`/`export_*`/`compact`/`build_wiki`.

## Topics to Explore

- [general] `reasons-vs-reasons_lib-duality` — Resolve why two near-identical packages exist (and why `merkle.py` is only in `reasons/`) before deep-reading either tree; this determines which files are canonical.
- [file] `pyproject.toml` — Locks down packages, entry points (`reasons.cli:main`, `reasons.mcp_server:main`), and the extras that map onto the repo's hidden subsystems (`pg`, `cluster`, `mcp`, `llm`, `vertex`, `openai`).
- [file] `README.md` — Defines the entire mental model (IN/OUT, SL/CP justifications, outlist, nogoods, dependency-directed backtracking, challenge/defend) that every core module implements.
- [file] `CLAUDE.md` — Repo-specific conventions; read before editing anything or trusting the tests.
- [file] `reasons/cli.py` — Primary entry point; the command dispatcher is the fastest index into every subsystem in the repo.
- [file] `reasons/storage.py` — The persistence schema for nodes, justifications, outlists, nogoods, dependents, and the audit log; all other modules sit on top of this.
- [file] `reasons/verify.py` — The atomic justification-validity check (SL/CP/outlist satisfaction); subtle bugs here corrupt every cascade and backtracking decision.
- [file] `reasons/network.py` — The propagation engine implementing Doyle's retraction cascade and restoration — the derivative center of the whole system.
- [file] `reasons/contradictions.py` — Nogood handling and dependency-directed backtracking: the hardest algorithm and the strongest differentiator of this TMS.
- [file] `reasons/derive.py` — Creates derived nodes with justifications; the seam between the pure TMS and the higher-level LLM-driven workflows (`ask`, `review`, `propose_update`).
- [file] `reasons/api.py` — Shared programmatic surface used by CLI, MCP server, and tests; establishes the type boundary for callers.
- [file] `reasons/merkle.py` — Integrity/hashing module present only in `reasons/`; a cheap probe for what the parallel `reasons_lib/` tree actually is.
- [file] `tests/conftest.py` — Fixtures and DB bootstrap; the fastest way to learn how storage/network are invoked and reset across the 60+ test files.
- [file] `tests/test_backtracking.py` — Authoritative behavioral spec of the hardest algorithm; compare against `contradictions.py` to lock in edge-case semantics.
- [file] `reasons/llm.py` — The LLM abstraction (`anthropic`/`vertex`/`openai`/`langfuse`) that every `review`, `repair`, `ask`, `propose_update`, and `import_agent` workflow depends on — the gateway to the "reasoning" half of the project.
