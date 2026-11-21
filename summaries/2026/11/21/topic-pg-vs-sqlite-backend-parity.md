---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The broader migration so many operations could drop `_require_sqlite`; explains the `*NoLongerBlocked` test classes and which commands remain SQLite-only.

# Broader migration: dropping `_require_sqlite`

The migration is moving CLI commands away from being SQLite-only toward backend-neutral API calls that can dispatch either to SQLite or PostgreSQL.

Two mechanisms make that possible:

- `_backend_kwargs` in `reasons/cli.py` selects backend arguments. The tests in `tests/test_pg_dispatch.py:TestBackendKwargs` show it returns `{"db_path": ...}` for the SQLite default, and `{"pg_conninfo": ..., "domain_id": ...}` when PostgreSQL is selected. It also exits if PostgreSQL is selected without a domain/project id.
- `_pg_dispatch` in `reasons/api.py` routes API calls into `reasons.pg.PgApi`. The tests at `tests/test_pg_dispatch.py:TestPgDispatch` patch `reasons.pg.PgApi` and verify that `_pg_dispatch` calls the corresponding PG API method with the expected kwargs.

Once a command’s underlying API operations have PostgreSQL paths, the CLI command no longer needs to call `_require_sqlite`. `_require_sqlite` is defined at `reasons/cli.py:1302`. Its behavior is covered by `tests/test_pg_dispatch.py` calls at lines 88, 93, and 99: with no PostgreSQL selection it passes; with `args.pg` set or `REASONS_PG_CONNINFO` set it raises `SystemExit`. So any command still calling `_require_sqlite` is explicitly rejecting PostgreSQL.

There are 24 total `_require_sqlite` matches in the observations: 1 definition, 4 test references, and 19 production call sites in `reasons/cli.py`. Those 19 call sites are the remaining SQLite-only command gates.

## What the `*NoLongerBlocked` tests mean

The three classes are in `tests/test_pg_dispatch.py`:

- `TestImportCliNoLongerBlocked` at line 335
- `TestMaintenanceCliNoLongerBlocked` at line 443
- `TestRepoAndNegativeCliNoLongerBlocked` at line 542

Their names indicate regression coverage for CLI command groups that used to be blocked by `_require_sqlite` but have since been migrated to support PostgreSQL dispatch. Based on the imports at the top of the file, those groups likely correspond to:

- Import-related operations: `import_json`, `import_beliefs`, `import_agent`, `sync_agent`
- Maintenance operations: `hash_sources`, `check_stale`, `lookup`
- Repo and negative-list operations: `add_repo`, `list_repos`, `list_negative`

The exact assertions inside those classes are not visible because the provided excerpt for `tests/test_pg_dispatch.py` stops at line 300, while the classes begin at lines 335, 443, and 542. So the missing information is the precise command names and mock targets each `*NoLongerBlocked` class asserts. However, the intent is clear: these are tests that the listed CLI command families no longer exit through the SQLite-only guard when PostgreSQL is configured.

Also note that `hash-sources` appears in the `_require_sqlite` tests at lines 88, 93, and 99 only as an arbitrary `command_name` label. `hash_sources` is imported from `reasons.api`, so that does not mean `hash-sources` remains SQLite-only.

## Commands that still remain SQLite-only

These are the production `_require_sqlite` call sites in `reasons/cli.py`, and therefore the commands still gated to SQLite:

| Line | Command |
|---:|---|
| 1156 | `check-integrity` |
| 1184 | `backfill-hashes` |
| 1191 | `pin-sources` |
| 1219 | `pin-update` |
| 1236 | `pin-lines` |
| 1355 | `ask` |
| 1387 | `cluster-list` |
| 1431 | `deduplicate` |
| 1664 | `derive` |
| 1771 | `accept` |
| 2321 | `build-wiki` |
| 2350 | `review-beliefs` |
| 2513 | `review-justifications` |
| 2578 | `review-premises` |
| 2690 | `repair-premises` |
| 2776 | `propose-update` |
| 2877 | `repair-smuggled` |
| 2919 | `repair` |
| 2969 | `detect-contradictions` |

The observations do not show the implementations of those commands or the full `PgApi` method surface, so they do not identify the exact technical blocker for each remaining command. Likely causes are missing PostgreSQL API methods, direct SQLite schema dependencies, or command-specific behavior that has not yet been ported.

## Topics to Explore

- [file] `tests/test_pg_dispatch.py` — Read the full `*NoLongerBlocked` classes at lines 335, 443, and 542; the provided excerpt stops at line 300.
- [function] `reasons/cli.py:_require_sqlite` — Confirm the exact exit message and argument/environment precedence used when PostgreSQL is selected.
- [function] `reasons/api.py:_pg_dispatch` — See how API-level calls instantiate `PgApi` and pass backend kwargs.
- [file] `reasons/cli.py` — Inspect the remaining `_require_sqlite` call sites and the command implementations around them.
- [general] `PgApi-method-coverage` — Compare `PgApi` methods against the 19 still-gated commands to identify what porting work remains.

## Beliefs

- `require-sqlite-gates-remaining-sqlite-only-commands` — The 19 production calls to `_require_sqlite` in `reasons/cli.py` are the commands that still reject PostgreSQL selection.
- `backend-kwargs-selects-sqlite-or-postgres` — `_backend_kwargs` returns `db_path` for default SQLite and `pg_conninfo`/`domain_id` for PostgreSQL selection.
- `pg-dispatch-enables-api-pg-paths` — `_pg_dispatch` and the API functions imported in `tests/test_pg_dispatch.py` provide PostgreSQL paths for import, maintenance, repo, and negative-list operations.
- `no-longer-blocked-tests-cover-migrated-cli-groups` — The three `*NoLongerBlocked` classes at `tests/test_pg_dispatch.py:335`, `:443`, and `:542` are regression tests for CLI groups that no longer use `_require_sqlite`.
- `hash-sources-string-is-test-only` — `hash-sources` appears in `_require_sqlite` tests only as a command label and is not among the remaining production `_require_sqlite` gates.
