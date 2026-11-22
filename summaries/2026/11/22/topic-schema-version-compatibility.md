---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Search for version comparisons, rejection paths, or upgrade hooks; current observations show none.

## Search for version comparisons, rejection paths, or upgrade hooks — what the observations actually show

Short answer: the *grep patterns* surfaced matches, but none of them implement **version comparison**, **version-gated rejection**, or an **automatic upgrade hook**. Every hit is either (a) domain vocabulary ("a newer/better version" of a belief), (b) a display-only read of the package version, or (c) a completely unrelated "upgrade" (truncated git hash → full hash). The one real migration is manual, dry-run-able, and not keyed on `schema_version`.

Below is the breakdown by search.

### 1. "version" — domain semantics + display metadata, no comparisons

- **Belief supersession chains** are the dominant meaning of "version" in this codebase: `reasons/api.py:1167` ("only the latest version is IN"), `reasons/api.py:1245`, `reasons/api.py:1286`, `reasons/network.py:438`, and the CLI help at `reasons/cli.py:3116` / `reasons/cli.py:3235`. This is the node supersession / duplicate-node model — not package or schema versioning.
- **Package version is read but never compared.** `reasons/cli.py:13` imports `importlib.metadata.version as _pkg_version`; `reasons/cli.py:3057` wires it into an argparse `action="version"` string. `reasons/metadata.py:17-18` does the same to populate metadata. In both cases the value is *emitted*, not used in a branch.
- **`pyproject.toml`** pins `version = "0.58.0"` and `requires-python = ">=3.10"` — the latter is enforced by pip at install time, not by any runtime check.
- **No semver parsing exists.** The `find_schema_version_refs` search for `packaging.version|version.parse|__version__` returned zero hits for those alternates; `pyproject.toml` also lists no `packaging` dependency. So there is no code path that can say "on-disk 1.0 < required 2.0."

### 2. "reject" — proposal workflow only, never version-based

- The only substantive rejection logic is `reasons/api.py:5801 reject_proposal`, which at `reasons/api.py:5819` runs `UPDATE proposals SET status = 'rejected'` and returns `{"status": "rejected"}` at `reasons/api.py:5840`. Docstring at `reasons/api.py:5809` is explicit: *"Mark a pending proposal as rejected. No truth change."*
- Wrappers: `reasons/cli.py:2023 cmd_reject_proposal`, subparser at `reasons/cli.py:3419-3422`, dispatch at `reasons/cli.py:3958`, MCP tool at `reasons/mcp_server.py:505`.
- `reasons/storage.py:108` is just a CHECK constraint enumerating the allowed statuses including `'rejected'`.
- The *closest* thing to a compatibility rejection is `reasons/api.py:3420-3437`, where unsupported filters (`challenged`, `min_depth`, `tag`, etc.) are collected and raise `"... not supported with PostgreSQL"`. That is a **backend capability gate**, not a version gate.
- Tests at `tests/test_api.py:1280-1297` confirm the semantics: rejection flips a proposal's status, so nothing is discarded based on data shape or version.

### 3. "upgrade" — git hash truncation, not schema/package upgrade

- `reasons/check_stale.py:66-157` defines `upgrade_hashes`, documented at lines 71-72 as: *"truncated hashes that are a prefix of the current full hash are upgraded in place."* Returns `(results, upgraded, sha_bumped)` at line 157.
- Surfaced through `reasons/api.py:2682-2726` (`upgrade_hashes: bool = False`, counters `{"upgraded": int, "sha_bumped": int}`), `reasons/pg.py:855-924` (batches `upgrades_to_commit`), and CLI at `reasons/cli.py:1119-1147` + flag at `reasons/cli.py:3519`.
- This is **opt-in, data-level normalization of commit SHAs**. Nothing here runs on DB open, and nothing here reads `schema_version`.

### 4. "migrat" — one explicit, version-agnostic migration

- `reasons/api.py:1549 migrate_retract_to_defeaters` converts string `retract_reason` metadata into graph-native defeater nodes (`defeater_type: "migrated-retraction"` at `reasons/api.py:1621`). Gated by `dry_run` at `reasons/api.py:1646`, returns `{"migrated", "skipped", "errors"}`.
- Invoked only manually: `reasons/cli.py:285 cmd_migrate_defeaters`, parser at `reasons/cli.py:3154-3156`, dispatch at `reasons/cli.py:3934`.
- Notably, `reasons/cli.py:287` hard-fails for `--pg`: *"migrate-defeaters is not supported with --pg (no PgApi implementation)."*
- `reasons/storage.py:136` has a comment about checking pre-existing tables "for migration," but that's a `CREATE IF NOT EXISTS`-style bootstrap, **not** a `schema_version`-keyed upgrade runner.
- `tests/test_migrate_defeaters.py:7` (basic) and `:34` (dry-run) confirm the API is only reached by explicit call — no autoload path.
- There is **no startup hook** that compares stored `meta.schema_version` against `SCHEMA_VERSION` and triggers this or any other migration.

### 5. `schema_version` is carried, not checked

- Written: `reasons/storage.py:195`, `reasons/pg.py:151`, `reasons/metadata.py:49` (hard-coded `SCHEMA_VERSION`).
- Backfilled: `reasons/storage.py:326` uses `meta.setdefault("schema_version", SCHEMA_VERSION)` — this *fills* a missing key, it does not *compare* an existing one.
- Passed through: `reasons/api.py:2127`, `reasons/pg.py:2250`, `reasons/pg.py:2398`, `reasons/import_beliefs.py:183` all iterate `("schema_version", "project_name", "created_at")` as carry-through fields.
- Emitted: `reasons/export_card.py:68`, `reasons/export_markdown.py:25`.
- The strongest negative evidence: `tests/test_import_hf.py:17` and `tests/test_hf_default_org.py:20` build fixtures with `"schema_version": "1.0"`, and `tests/test_metadata.py:134-136` imports a hand-authored YAML front matter with `schema_version: "1.0"` — all succeed. A `"1.0"` payload is accepted while the package declares `0.58.0`, proving no compatibility enforcement on import.

### What this means for the colleague

The rejection path you'd expect for a stale on-disk format (e.g. "refuse to open a DB whose `schema_version` doesn't match `SCHEMA_VERSION`") **does not exist**. The only things guarded are:
1. Proposal lifecycle state (`reject_proposal`), and
2. PostgreSQL feature coverage (`reasons/api.py:3435-3437`).

Everything else accepts whatever `schema_version` string it is handed and relies on **manual** CLI commands (`reasons migrate-defeaters`, `reasons check-stale --upgrade-hashes`) to fix data.

### What's still missing from the observations

- The **definition** of `SCHEMA_VERSION` (used at `reasons/metadata.py:49`, `reasons/storage.py:195`/`:326`, `reasons/pg.py:151`) is not in this dump. It's almost certainly in `reasons/metadata.py` just above line 17, but that line wasn't captured.
- `reasons/init_db` in `reasons/storage.py` (~lines 100-330) and the PG equivalent (~`reasons/pg.py`:100-230) were only seen by reference — a full read is needed to definitively rule out a "if stored_version != SCHEMA_VERSION: migrate()" branch.
- `reasons/import_beliefs.py` was only seen at the carry-through loop; the full loader wasn't read, so a `try/except` on incompatible input cannot be fully excluded.
- `reasons/network.py` meta loading/saving path (only `line 25` and `line 438` surfaced) needs a read to confirm `meta` is never validated.

Until those four reads are in, the conclusion is "no evidence of comparisons/rejection hooks in what was searched," corroborated strongly by the absence of `packaging`/`version.parse` and by the `"1.0"` test fixtures being accepted.

## Topics to Explore

- [file] `reasons/metadata.py` — Locate the `SCHEMA_VERSION` constant and confirm whether it is ever compared, or only written into metadata.
- [function] `reasons/storage.py:init_db` — Confirm DB bootstrap does not branch on the stored `meta.schema_version`; `storage.py:136` references pre-existing-table checks.
- [file] `reasons/import_beliefs.py` — Verify that importing a file with an older/unknown `schema_version` never raises or warns.
- [function] `reasons/pg.py:check_stale` — Contrast the hash-upgrade path with a hypothetical schema-upgrade path; confirm the two are unrelated.
- [general] `schema-version-compatibility-gap` — Decide whether to add a real version gate (compare `meta["schema_version"]` to `SCHEMA_VERSION` on open/import) or document that format drift is unsupported by design.

## Beliefs

- `schema-version-is-write-only` — `schema_version` is written to storage and exports (`reasons/storage.py:195`, `reasons/storage.py:326`, `reasons/export_card.py:68`) and carried through imports (`reasons/import_beliefs.py:183`), but no code path compares a stored value to `SCHEMA_VERSION`.
- `runtime-version-is-display-only` — `importlib.metadata.version` is used only to render `--version` output (`reasons/cli.py:3057`) and metadata (`reasons/metadata.py:18`); the result never feeds a conditional.
- `rejection-is-proposal-scoped` — The only "reject" implementation is `reasons/api.py:reject_proposal` (line 5801), which mutates `proposals.status` to `'rejected'`; no rejection path is triggered by version or schema mismatch.
- `upgrade-hashes-is-not-schema-upgrade` — `reasons/check_stale.py:66` `upgrade_hashes` expands truncated git commit SHAs to full length and is unrelated to package or schema upgrades.
- `postgres-capability-gate-is-the-only-compat-rejection` — `reasons/api.py:3420-3437` is the sole code path that raises on an unsupported request, and it gates on PostgreSQL backend capability (`tag`, `min_depth`, `challenged`, etc.), not on version.
- `import-accepts-mismatched-schema-version` — `tests/test_import_hf.py:17` and `tests/test_metadata.py:134-136` import fixtures declaring `schema_version: "1.0"` while `pyproject.toml` declares `0.58.0`, and the import succeeds, demonstrating no compatibility check on load.
