---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Worth tracing how the CLI and API resolve database paths, since this function claims to match CLI behavior.

# Tracing "CLI-matching" DB path resolution

## Bottom line: the claim isn't verifiable from what we were given

Both code searches came back empty:

- `locate_cli_behavior_claim` → 0 matches for anything like "mirrors / matches / same as … CLI"
- `find_db_path_resolution_symbols` → 0 matches for `db_path`/`database_url`/`resolve.*path` style symbols

That's suspicious, because **both files definitely contain these symbols** — we can see `DEFAULT_DB = "reasons.db"` at roughly `reasons/api.py:30`, `db_path: str = DEFAULT_DB` in dozens of signatures, and `args.db` used throughout `reasons/cli.py`. So the empty results mean one of two things:

1. The patterns were applied only to the 600-line excerpts that were retrieved, not the full files (`reasons/api.py` is 5,886 lines; `reasons/cli.py` is 4,015 lines — we're seeing ~10–15% of each), **or**
2. The function that "claims to match CLI behavior" lives in the un-retrieved portion, so we never actually saw it.

Either way, the specific function making the claim has not been observed. What we *can* do is map the two resolution paths that are visible and identify exactly where they could diverge.

## What the API does (visible)

`reasons/api.py` resolves paths in three layers:

1. **Module default** — `DEFAULT_DB = "reasons.db"` (≈`api.py:30`). A bare relative filename, so it resolves against the process CWD. There is no `Path.home()`, no XDG dir, no env-var lookup at module scope.
2. **Per-function default argument** — e.g. `init_db(db_path: str = DEFAULT_DB, …)`, `ensure_namespace(…, db_path: str = DEFAULT_DB, …)`, `add_node(…, db_path: str = DEFAULT_DB, …)`, `retract_node(…, db_path: str = DEFAULT_DB, …)`, `what_if_retract(…, db_path: str = DEFAULT_DB, …)`. Python binds these defaults **at import time**, so `DEFAULT_DB` is frozen into every signature.
3. **Backend switch** — every function begins with `if pg_conninfo: return _pg_dispatch(pg_conninfo, domain_id, "<method>", …)`. So `db_path` is only consulted when `pg_conninfo` is falsy; the SQLite path and the Postgres domain are mutually exclusive branches.

Downstream, `reasons/storage.py:Storage.__init__` does `self.db_path = Path(db_path)` and `sqlite3.connect(str(self.db_path))`. `init_db` similarly does `p = Path(db_path)` and only unlinks on `force`. **Neither creates parent directories** — so a caller passing `~/.config/reasons.db` will get a literal `~` directory if the shell didn't expand it first.

Cache subtlety: `_with_network` keys `_network_cache` on the **raw `db_path` string**. `"reasons.db"`, `"./reasons.db"`, and `"/abs/path/reasons.db"` are three distinct cache entries pointing at the same file, and stale-read behavior differs across them until mtime changes.

## What the CLI does (partially visible)

`reasons/cli.py` has **two inconsistent idioms**, and this is the real finding:

**Idiom A — arg-bag delegation (`_backend_kwargs`)**, used by `cmd_init`, `cmd_add`, `cmd_add_justification`, `cmd_remove_justification`, `cmd_retract`, `cmd_assert`, `cmd_what_if`, `cmd_status`, `cmd_show`, `cmd_tag`, `cmd_untag`:

```python
result = api.add_node(..., **_backend_kwargs(args))
```

Note that `cmd_init` (cli.py ≈line 16–28) does **not** pass `db_path` at all — it relies entirely on `_backend_kwargs(args)` to inject `db_path`/`pg_conninfo`/`domain_id`. If `_backend_kwargs` omits `db_path`, `cmd_init` falls through to `api.DEFAULT_DB`.

**Idiom B — explicit `args.db` plus manual backend guard**, used by `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, `cmd_classify_defeaters`:

```python
if getattr(args, "pg", None) or os.environ.get("REASONS_PG_CONNINFO"):
    ...sys.exit(1)
result = api.mark_duplicate(args.source_id, args.canonical_id, db_path=args.db)
```

These bypass `_backend_kwargs` entirely — no `pg_conninfo`, no `domain_id`, and the env-var check is duplicated inline.

## Where the claim can break

The claim under investigation ("this function matches CLI behavior") is credible only if the function in question replicates **Idiom A**, because Idiom B is a different contract. Concretely, any code that reproduces `DEFAULT_DB` or `args.db` but **not** `_backend_kwargs` will:

- miss whatever env-var / config-file / `--db` precedence `_backend_kwargs` implements;
- silently default to a CWD-relative `reasons.db` when the CLI would have opened a different file;
- skip the `pg_conninfo` branch entirely, so a CLI invocation under `REASONS_PG_CONNINFO` hits Postgres while the "matching" API call hits SQLite.

## Topics to Explore

- [function] `reasons/cli.py:_backend_kwargs` — The single unseen function that decides `db_path` / `pg_conninfo` / `domain_id` for Idiom-A commands; until you read it, "matches CLI behavior" cannot be confirmed or refuted.
- [function] `reasons/cli.py:main` — Argparse wiring, where `--db`, `--pg`, and any `REASONS_*` env-var defaults are declared; this is where the actual precedence order lives.
- [function] `reasons/api.py:init_db` — The only API entry point that does its own `Path(db_path)` existence/`force` handling; a good oracle for how a relative `DEFAULT_DB` behaves on first run.
- [function] `reasons/api.py:_with_network` — Mtime-keyed cache keyed on the raw non-normalized path string; explains path-identity bugs that are invisible at the CLI layer.
- [general] `db-path-normalization` — Whether `Path.resolve()`, `expanduser()`, or parent-dir creation exists anywhere in the 3,400–5,300 unretrieved lines of api.py/cli.py; the excerpts show none.

## Beliefs

- `DEFAULT_DB-is-a-relative-filename` — `reasons/api.py` defines `DEFAULT_DB = "reasons.db"` with no expanduser/resolve, so every defaulted API call resolves against the caller's CWD.
- `api-functions-take-db_path-default-argument` — Nearly all public API functions declare `db_path: str = DEFAULT_DB`, binding the default at import time rather than reading config per call.
- `pg_conninfo-preempts-db_path` — Every API function checks `if pg_conninfo:` and dispatches to `_pg_dispatch` before touching `db_path`, so SQLite and Postgres paths are mutually exclusive by construction.
- `cli-has-two-db-resolution-idioms` — `cmd_add`/`cmd_retract`/`cmd_init` delegate to `_backend_kwargs(args)`, while `cmd_mark_duplicate`/`cmd_migrate_defeaters` pass `db_path=args.db` directly and guard on `REASONS_PG_CONNINFO` inline.
- `network-cache-key-is-unnormalized-db_path` — `_network_cache` in `reasons/api.py` is keyed on the raw `db_path` string, so different spellings of the same file produce separate cache entries.
- `storage-does-not-create-parent-directories` — `Storage.__init__` in `reasons/storage.py` calls `sqlite3.connect(str(Path(db_path)))` without `mkdir(parents=True)`, so a missing parent directory raises at connect time.
