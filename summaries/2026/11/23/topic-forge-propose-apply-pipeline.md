---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: How `.forge/proposed-entries.json` is consumed to materialize `entries/` (the `entries/` tree is entirely absent from this checkout, so the apply step lives elsewhere — likely `.github/workflows/` or a separate wiki repo).

## How `.forge/proposed-entries.json` is consumed — short answer: **it isn't, in this checkout**

The observations do not contain the consumer. Every search designed to find it comes back empty:

- `find_references_to_proposed_entries` (pattern `proposed-entries|proposed_entries|\.forge`) → **0 matches** across the repo.
- `find_entry_materialization_in_python` (pattern `entries/|apply_proposed|apply_entries|materialize|write_text(.*entries|mkdir(.*entries`) → **10 matches, all in `tests/`**; none are writers of `entries/`. They are tests that *read* or *reference* `entries/...` paths for `resolve_source_path` (`tests/test_check_stale.py`) and for source-string assertions (`tests/test_backtracking.py:238`).
- `.github/workflows/` contains exactly one file, `tests.yml` — but `read_all_workflow_lines` returned **0 lines / 0 matches**, so we effectively cannot see its contents. That tool result is suspicious (a `.` regex against a non-empty file should match every line); treat the workflow as *unverified*, not *confirmed absent of the applier*.

So: given the evidence, **the apply step is not in this repository.** It is downstream (a `.forge/` CLI tool, a scheduled job in a different repo, or a hand-run command that clones/copies `entries/`).

## What we *can* say about the file

`.forge/proposed-entries.json` is a flat JSON object — not a list of rich proposal records like the ones `propose_update.py` generates:

```json
{
  "entries/2026/04/23/reasons_lib-__init__.md": "9a70f072a5d29dbd",
  "entries/2026/04/23/reasons_lib-api.md": "be57ef94e9958794",
  ...
}
```

Shape: **17 keys → 16-hex-char content hashes.** Every key is a path under `entries/YYYY/MM/DD/<slug>.md`. The dates are all `2026/04/23` — i.e., a single batch of generated entry files, one per source module / topic (`reasons_lib-api`, `reasons_lib-cli`, `topic-multi-agent-federation`, `scan-ftl-reasons`, …).

This maps cleanly onto the `source` + `source_hash` pattern used throughout the TMS network. In `tests/test_check_stale.py` we see exactly this shape:

- line 108: `net.add_node("a", "Premise A", source="entries/topic.md", source_hash=h)`
- line 203 / 222: `source="entries/topic.md", source_hash=h,`
- line 53: `resolve_source_path("entries/2026/topic.md", db_dir=tmp_path)`
- line 63: `resolve_source_path("entries/topic.md", repos={"entries": repo_dir}, db_dir=db_dir)`

So the semantic of `proposed-entries.json` is best read as: **a manifest of "these files should exist at these paths with this content, with this hash"** — exactly the pair (`source` path, `source_hash`) that `check_stale` later verifies. The applier must **materialize each path into an `entries` repo and record the hash so the network's staleness check can validate it.**

## Why the `reasons/` modules you were shown are *not* the consumer

- `reasons/propose_update.py` — despite the name, it produces **belief-update proposals** (objects with `id`, `action: update|retract`, `failure_mode`, `basis`, `evidence`, `comment` — see `parse_update_proposals` around the middle of the file). Not file materializations. It never writes `.forge/proposed-entries.json` and never reads `entries/`.
- `reasons/review.py` — audits derived beliefs. Irrelevant to entry files.
- `reasons/build_wiki.py:build_wiki` — writes markdown (`index.md` + per-group pages) to a caller-supplied `output_dir`. This is the wiki generator, not the entries generator. If `entries/` and `wiki/` are separate trees, they are written by separate tools.

`reasons/propose_update.py` and `.forge/proposed-entries.json` are **adjacent but unrelated**: both are "propose X" flows, but the JSON is a plain path→hash map, whereas the belief proposal pipeline emits rich objects. Do not confuse the two when reading the code.

## The one place the code *knows* about `entries/`

`resolve_source_path` (exercised by `tests/test_check_stale.py:53-84`) resolves a source string like `entries/2026/topic.md` against a **repo mapping** — `resolve_source_path("entries/topic.md", repos={"entries": repo_dir}, db_dir=db_dir)` — with a fallback to `db_dir` and pinning to an `agent` (e.g. `agent="code"`). This is the strongest structural hint about the consumer:

- `entries` is a **named repo** in the config, resolved like `{"entries": <checkout-or-remote>}`.
- Nodes are keyed by `(source_path, source_hash)`, and the proposed JSON is the source of that hash.
- Therefore the applier most plausibly: reads `.forge/proposed-entries.json` → for each `path` → clones/updates the `entries` repo → writes file contents whose hash matches → the resulting tree is then consumable by `check_stale`.

## What's missing (and where to look next)

1. **`.github/workflows/tests.yml`** — the only workflow, but the tool read returned nothing. Even if it only runs tests here, its contents decide whether the applier is invoked on PR/merge or not. Re-read it directly.
2. **A separate `entries` (or `wiki`) repository** — `resolve_source_path(..., repos={"entries": ...})` strongly implies one exists. The applier probably lives there, and `.forge/proposed-entries.json` is committed into *this* repo as the hand-off artifact.
3. **The `.forge/` directory listing** — only `proposed-entries.json` was sampled. Other files (e.g., a `Makefile`, config, or a lockfile) would reveal the writer.
4. **`git log`** on `.forge/proposed-entries.json` — commits that add entries to this JSON are almost certainly accompanied by commits to the `entries` repo (submodule bump, PR, or CI bot commit).

Bottom line for a colleague: **do not look for the consumer in this checkout — it is not here.** The JSON is a self-describing manifest of intended `entries/` files (path → content hash) that matches the `source` + `source_hash` model used by `check_stale.resolve_source_path`. The materializer is external; `reasons/propose_update.py` and `reasons/build_wiki.py` are *not* it.

## Topics to Explore

- [file] `.github/workflows/tests.yml` — The only workflow file; the tool returned no lines, so its contents are unverified and it's the top candidate for holding the apply step.
- [function] `reasons/check_stale.py:resolve_source_path` — Shows how `entries/...` paths are resolved against a `repos` mapping and `agent`; reveals the expected layout of the `entries` repo the applier must produce.
- [repo] `entries` (external) — `tests/test_check_stale.py:63` treats `entries` as a named repo; the materializer almost certainly lives in or targets that repo.
- [file] `.forge/` (directory listing) — Enumerate siblings of `proposed-entries.json` to find what writes it (a tool, config, or lockfile).
- [general] `git-history-for-proposed-entries` — `git log -p -- .forge/proposed-entries.json` should pair each JSON update with the corresponding `entries/` commit.

## Beliefs

- `no-code-consumes-proposed-entries` — No source file in this checkout references `proposed-entries`, `.forge`, or `proposed_entries`; the apply step lives outside this repository.
- `proposed-entries-is-a-path-to-hash-manifest` — `.forge/proposed-entries.json` is a flat object mapping `entries/YYYY/MM/DD/<slug>.md` paths to 16-hex-char content hashes (17 entries, all dated `2026/04/23`).
- `propose-update-module-targets-beliefs-not-entries` — `reasons/propose_update.py` emits belief-update objects (`id`, `action`, `failure_mode`, `basis`) and has no code path that reads or writes `.forge/proposed-entries.json`.
- `build-wiki-writes-to-output-dir-not-to-entries` — `reasons/build_wiki.py:build_wiki` writes `index.md` and per-group pages into its `output_dir` argument, not into `entries/`.
- `entries-is-external-named-repo` — `tests/test_check_stale.py` calls `resolve_source_path("entries/...", repos={"entries": repo_dir}, agent="code")`, showing `entries` is resolved as a named external repo rather than a path inside this checkout.
