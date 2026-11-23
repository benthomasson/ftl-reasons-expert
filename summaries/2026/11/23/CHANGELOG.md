---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# File: CHANGELOG.md

# `CHANGELOG.md` — Explanation

## Purpose

`CHANGELOG.md` is the **curated, human-readable release history** for the `reasons` (PyPI: `ftl-reasons`) truth-maintenance-system (TMS) library. Unlike most Python files in this repo, it is not executable — it is a **chronological product spec** that:

1. Records every notable change per semantic version (0.1.0 → 0.18.0, dated 2026‑03‑23 → 2026‑04‑25).
2. Acts as the canonical **what-shape-was-the-product-at-each-release** artifact used by tooling like `build_wiki.py`, the `.forge` proposal pipeline, and (per 0.3.0) Python-LLM consumers that "read the repo to learn it".
3. Serves as the release boundary marker — the topmost `## [x.y.z]` header is expected to equal the package version in `pyproject.toml`.

It is **not** auto-generated from git log; entries are behavior-level prose ("Retract/assert cascade output split into 'Went OUT' and 'Went IN' sections (#7)"), not commit hashes.

---

## Key Components

The file is a single, flat document. Its "components" are its **structural elements**, each of which a maintainer must respect:

| Element | Format | Example | Contract |
|---|---|---|---|
| Preamble | Two lines | `All notable changes…` / `The format is based on Keep a Changelog…` | Must reference spec version (1.1.0) |
| Version header | `## [x.y.z] - YYYY-MM-DD` | `## [0.18.0] - 2026-04-25` | Reverse chronological; brackets mandatory |
| Section header | `### Added` / `### Changed` / `### Fixed` | `### Added` | A version may have 1–3 sections, never zero |
| Entry | `- <sentence> (#N[, PR #M])` | `- Access tags (\`access_tags\`) for data source provenance filtering (RBAC) (#38, PR #42)` | Imperative-past tense, optional issue/PR cross-ref |
| Cross-ref | `(#N)`, `(PR #N)`, `(#N, PR #M)` | `(#11, PR #14)` | Enables traceability to `tests/test_*.py` and `.github/workflows` |

There are **no functions, classes, or constants** — but there are two implicit constants the maintainers must keep in sync:

- `LATEST_VERSION = "0.18.0"` (top header)
- `LATEST_DATE = "2026-04-25"`
- Both must match `pyproject.toml` `[project].version`.

### Release narrative (what each version means operationally)

- **0.1.0** — Core TMS: nodes, SL justifications, truth propagation, outlist (non-monotonic), dependency-directed backtracking, `challenge`/`defend`, SQLite persistence, `reasons_lib.api`, full initial CLI.
- **0.2.0** — Packaging (PyPI) + `lookup`, FTS5, `convert-to-premise`, `import-json`, summarization justifications.
- **0.3.0** — **Rebrand** `rms` → `reasons`, motivated by a 5pp LLM-accuracy ablation. PyPI name `ftl-reasons`.
- **0.4.0** — `supersede`, `--reason` on `retract`, repos table.
- **0.5.0 / 0.6.0** — `what-if` (read-only retraction/assertion simulation).
- **0.7.0** — `derive` (LLM-proposed deeper chains) + `import-agent` (multi-agent namespaces).
- **0.10.0** — `accept` (apply derive proposals), plus features attributed to 0.8/0.9 (`--exhaust`, topic/budget/random sampling).
- **0.12.0** — `deduplicate` (Jaccard similarity).
- **0.13.0** — CI, retract/assert cascade splitting, variant-ID guard.
- **0.14.0** — import-agent outlist fix, JSON outlist validation, fixpoint `recompute_all`.
- **0.15.0** — depth filters (`--min-depth`, `--max-depth`, `--premises`, `--has-dependents`).
- **0.16.0** — `_retracted` flag correctness, metadata-key scrubbing on JSON export.
- **0.17.0** — `add-justification`, `--any` (OR-mode), `sync-agent`.
- **0.18.0** — **Access-control layer**: `access_tags`, `--visible-to`, `trace-access-tags`, `PermissionError`, nogood filtering, +63 CLI tests (27 commands).

---

## Patterns

1. **Keep a Changelog 1.1.0** — overridden only in that no `Unreleased` heading is used (see Invariants).
2. **Semantic-ish versioning** — every release bumps the minor and keeps `.0` patch; there has never been a patch release. Deprecation of a public name (`rms` → `reasons`) was done inside a **minor** bump (0.3.0), not a major — pre‑1.0 convention.
3. **Story-arc releases** — each minor version is thematically coherent (e.g., 0.4.0 = supersession; 0.18.0 = RBAC). A maintainer should **not** mix unrelated features in a minor bump.
4. **Behavior-level changelog** — entries describe user-facing effects ("`PermissionError` on single-node queries (`show`, `explain`, `trace`)"), not file paths or PR titles. This makes the file readable by LLM agents (which the project explicitly targets).
5. **Issue/PR annotation** — `(#N)` for issues, `(PR #M)` for PRs, both when useful. This is how you cross-link from `CHANGELOG.md` into `tests/test_*.py`.
6. **Hidden-version references** — 0.10.0 cites "(0.9.0)" and "(0.8.0)" in prose without providing headers. Treat these as historical artifacts of squashed releases.
7. **Top-level metadata scrub** — 0.16.0 documents that `_retracted` and other `_`-prefixed internal keys **never** appear in JSON export. This is a **contract** downstream consumers rely on.

---

## Dependencies

**Static imports:** none. `CHANGELOG.md` is not code and cannot import anything.

**Implicit upstream dependencies (must stay consistent):**

- `pyproject.toml` — `[project].version` must equal the top header (`0.18.0`).
- `README.md` — often summarises the current release; drift between README and CHANGELOG is a common failure mode.
- `.forge/proposed-entries.json` and `.forge/report.json` — machine-generated changelog proposals that presumably feed the human-curated entries.
- `.github/workflows/*` — likely runs on push/PR (added in 0.13.0). A CI gate that fails when `pyproject.toml` version ≠ top `CHANGELOG` header would be the natural place to enforce sync.
- `reasons/cli.py` — the authoritative list of implemented commands; 0.18.0 states **27 commands** covered by 63 CLI tests. Any new CLI command must appear here.

**Implicit downstream consumers:**

- `reasons/build_wiki.py` — likely renders the changelog into a wiki.
- `reasons/export_markdown.py` and `compact.py` — per 0.4.0, `retract --reason` text appears in their output; the changelog documents that behavior contract.
- LLM agents (per 0.3.0's ablation motivation) that use the changelog to infer feature availability by version.
- PyPI project page (the "long description" often includes CHANGELOG).

---

## Flow

There is no runtime flow — but there is a **release flow** that produces the file:

```
 PR merged
   ↓
 .forge proposal (proposed-entries.json) ← automated diff-based suggestion
   ↓
 maintainer edits CHANGELOG.md on release branch
   ↓
 bump pyproject.toml version
   ↓
 open new `## [x.y.z] - YYYY-MM-DD` header at TOP
   ↓
 move Unreleased contents under that header (no explicit Unreleased block is kept)
   ↓
 tag + publish to PyPI (as `ftl-reasons`)
```

Practically: **read top-down to learn history; read bottom-up to see evolution**. Days with multiple releases (2026-03-23 has four versions; 2026-04-17 has four) indicate **rapid iteration during development**, not semver violations.

---

## Invariants

A changelog edit must preserve these:

1. **Header format is exact** — `## [x.y.z] - YYYY-MM-DD` with brackets. Any tooling parsing this file (wiki builder, `.forge` diff) will break if brackets or date are missing.
2. **Reverse chronology** — Highest version first. Dates are non-decreasing as you read down.
3. **Version monotonicity** — No header may repeat. Currently: `0.1.0 … 0.7.0`, then `0.10.0`, `0.12.0–0.18.0`. **Gap-versions 0.8.0, 0.9.0, 0.11.0 have no headers** — do not "fix" this without understanding why; the 0.10.0 body explicitly attributes features to 0.8.0/0.9.0, implying those were folded in.
4. **No empty releases** — every `##` header must be followed by at least one `###`.
5. **Only `Added` / `Changed` / `Fixed`** are used so far. `Removed`, `Deprecated`, `Security` are valid Keep-a-Changelog sections but are currently unused; introducing them changes the taxonomy consumers may expect.
6. **No `Unreleased` block.** Work in progress is tracked in `.forge/`, not in this file. If you start the next release, do **not** add an `[Unreleased]` header without updating downstream parsers.
7. **`_`-prefixed metadata never appears in exported JSON** (0.16.0). This is a behavioral contract the changelog itself records.
8. **Version in `pyproject.toml` must match the top header.** This is the single most likely invariant to be silently broken in CI.

---

## Error Handling

The file itself cannot raise. But the changelog **documents three error-handling semantics** that are contracts the code must honor:

- **`PermissionError`** — 0.18.0: "`PermissionError` on single-node queries (`show`, `explain`, `trace`) when access denied." Read commands therefore raise, not return an empty result, when the caller's `--visible-to` doesn't overlap `access_tags`.
- **Precondition guard** — 0.13.0: "Accept/derive guards against re-introducing retracted beliefs under variant IDs (#5)." `accept` and `derive` must reject entries whose canonical IDs collide with a `_retracted` node.
- **Retraction reason preservation** — 0.4.0: reasons must survive into `export-markdown` and `compact` output. Silent loss of the reason is a regression.

There are **no documented error contracts for the write path** in this file (e.g., what `add` does when a node ID already exists) — that's a gap a maintainer should look for in `reasons/cli.py` and `reasons/api.py`.

---

## Topics to Explore

- [file] `pyproject.toml` — Confirm `[project].version` equals the `0.18.0` header; this is the top invariant and most likely drift point.
- [file] `.forge/proposed-entries.json` — The automated proposal feed that merchants this CHANGELOG; reveals how entries are generated and reviewed.
- [file] `reasons/cli.py` — Source of the 27-command surface asserted in 0.18.0; verify every documented command (`trace-access-tags`, `sync-agent`, `accept`, `deduplicate`, `what-if`, …) is actually present.
- [file] `tests/test_access_tags.py` — The 63-test suite referenced in 0.18.0 backing the RBAC feature; the contract `PermissionError` on `show`/`explain`/`trace` should appear here.
- [general] `release-process-and-versioning` — How `pyproject.toml`, `CHANGELOG.md`, `.forge/`, and `.github/workflows` coordinate version bumps; explains missing 0.8.0/0.9.0/0.11.0 headers.
- [file] `reasons/api.py` — The "Python API layer returning dicts for CLI and tool-call consumers" (from 0.1.0); the dict shape is a public contract.

---

## Beliefs

- `changelog-uses-keep-a-changelog-format` — The file conforms to Keep a Changelog 1.1.0, with only `Added`/`Changed`/`Fixed` sections used so far.
- `changelog-latest-version-is-0-18-0` — The most recent documented release is `0.18.0` dated `2026-04-25`; `pyproject.toml` version must equal this.
- `package-renamed-to-reasons-in-0-3-0` — The project was renamed from `rms` to `reasons` (PyPI `ftl-reasons`) in 0.3.0, justified by a 5pp LLM-accuracy ablation.
- `changelog-omits-0-8-0-0-9-0-0-11-0-headers` — Versions 0.8.0, 0.9.0, and 0.11.0 have no `##` headers; 0.8.0/0.9.0 features appear folded into the 0.10.0 entry.
- `access-tags-implement-rbac` — 0.18.0 added `access_tags` provenance filtering; single-node queries (`show`, `explain`, `trace`) raise `PermissionError` when denied, and derived nodes inherit the union of parent tags.
- `cli-surface-has-27-commands-in-0-18-0` — 0.18.0 documents 63 CLI tests covering 27 commands, establishing `reasons/cli.py` as the authoritative command list.
- `internal-metadata-never-appears-in-json-export` — Per 0.16.0, `_`-prefixed keys like `_retracted` are scrubbed from JSON export; this is a downstream contract.
