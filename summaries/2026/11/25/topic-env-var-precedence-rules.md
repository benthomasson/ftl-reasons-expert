---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The specific ordering of `--domain-id > --project-id > REASONS_DOMAIN_ID > REASONS_PROJECT_ID` and the implication for tooling that populates both.

## Resolution Ordering in `reasons/cli.py`

### What the code actually shows

Two pieces of evidence establish the ordering.

**1. The parser declarations** (`reasons/cli.py:3061–3064`):

```python
parser.add_argument("--domain-id", default=None,
                    help="Domain ID for PostgreSQL (or set REASONS_DOMAIN_ID)")
parser.add_argument("--project-id", default=None,
                    help="(deprecated alias for --domain-id)")
```

Both flags default to `None`, and `--project-id` is explicitly documented as a **deprecated alias** of `--domain-id`.

**2. The resolution expression** (`reasons/cli.py:1292–1293`):

```python
        or os.environ.get("REASONS_DOMAIN_ID")
        or os.environ.get("REASONS_PROJECT_ID"))
```

The leading `or` on line 1292 means this is the tail of a single short-circuit expression. Combined with the stated premise, the effective logic is:

```python
domain_id = (
    args.domain_id                          # 1. --domain-id
    or args.project_id                      # 2. --project-id (deprecated)
    or os.environ.get("REASONS_DOMAIN_ID")  # 3. modern env var
    or os.environ.get("REASONS_PROJECT_ID") # 4. legacy env var
)
```

Because Python's `or` returns the first **truthy** value, every tier is only consulted if each higher tier is `None` or `""`. There is no merging and no conflict detection — it is a strict pick-the-first-non-empty chain.

Line 1296 then enforces the outcome when the PG backend is selected:

```python
print("Error: --domain-id is required with --pg", file=sys.stderr)
```

So if the **entire chain** resolves to a falsy value while `--pg` is active, the CLI aborts.

> **Caveat:** The observation excerpt for `reasons/cli.py` returned only lines 1–500, so lines 1290–1291 (the `args.domain_id` / `args.project_id` part of the chain) are not directly visible. The ordering of the CLI flags is inferred from the fragment at 1292–1293 plus the deprecated-alias help text at 3064.

### Implications for tooling that populates both

1. **Populating both layers is safe but redundant.** If a wrapper sets `--domain-id` *and* `--project-id`, or `REASONS_DOMAIN_ID` *and* `REASONS_PROJECT_ID`, the higher-precedence value silently wins. There is no warning that the lower tier was ignored, so a tool cannot detect a misconfiguration from stdout/stderr alone.

2. **Conflicting values will not error.** `REASONS_DOMAIN_ID=foo` alongside `REASONS_PROJECT_ID=bar` resolves to `foo`; the `bar` value is discarded. Tooling that populates both "for safety" should ensure they are identical, otherwise debugging a domain mismatch requires re-reading the `or` chain.

3. **Legacy compatibility is intentional.** `tests/test_pg_dispatch.py` explicitly sets `"REASONS_PROJECT_ID": "env-proj"` (lines 68 and 76) and treats it as a real env input (excluded from strips at lines 44 and 60). So `REASONS_PROJECT_ID` is still exercised and is not dead code — tooling can keep emitting it during a migration window, but should emit `--domain-id` / `REASONS_DOMAIN_ID` when feasible per the deprecation note at line 3064.

4. **Empty strings fall through.** Because `or` treats `""` as falsy, a tool that passes `--domain-id ""` to mean "unset" will actually trigger the lower tiers. If a tool emits `--project-id` as a placeholder, an empty `--domain-id` gives the placeholder a chance to win.

5. **Test coverage gap.** `find_precedence_tests` returned **zero** matches for patterns pairing `domain_id`/`project_id` or `REASONS_DOMAIN_ID`/`REASONS_PROJECT_ID`. There is no test asserting the declared order, so a refactor that swaps two `or` operands would not be caught. Tooling that depends on this ordering has no regression guard today.

6. **The error message names only the canonical flag.** Line 1296 says `--domain-id is required` even though `--project-id` is an accepted alternative. Tool authors reading that message should still migrate to `--domain-id`, since `--project-id` is the deprecated surface.

### What's missing from the observations

- Lines 1290–1291 (the `args.domain_id` / `args.project_id` portion of the chain) — needed to confirm the CLI-flag half of the ordering exactly.
- `_backend_kwargs` (referenced in `cmd_init`, `cmd_add`, etc.) — needed to see whether the resolved `domain_id` is re-validated or re-ordered before hitting `api`/`storage`.
- The context of the `--pg` check at line 1296 — whether it runs before or after `_backend_kwargs` is built.

## Topics to Explore

- [file] `reasons/cli.py` — Read lines 1280–1300 in full to confirm the precise `or` chain and the `--pg` validation order.
- [function] `reasons/cli.py:_backend_kwargs` — Determine how the resolved domain ID is forwarded to `api` calls and whether any second precedence rule exists there.
- [file] `tests/test_pg_dispatch.py` — Existing PG dispatch tests (lines 44, 60, 68, 76) show how `REASONS_PROJECT_ID` is exercised today.
- [general] `precedence-test-gap` — No tests assert `--domain-id` vs `--project-id` or `REASONS_DOMAIN_ID` vs `REASONS_PROJECT_ID` ordering; consider adding one.
- [file] `reasons/storage.py` — See how the resolved `domain_id` is consumed by the PostgreSQL backend and whether it is re-validated.

## Beliefs

- `domain-id-precedence-chain` — At `reasons/cli.py:1291–1293`, domain ID resolution is a single short-circuit `or` chain ordered `--domain-id > --project-id > REASONS_DOMAIN_ID > REASONS_PROJECT_ID`.
- `project-id-is-deprecated-alias` — `--project-id` is declared at `reasons/cli.py:3063` with the help string "(deprecated alias for --domain-id)".
- `domain-id-required-with-pg` — `reasons/cli.py:1296` aborts with "Error: --domain-id is required with --pg" when the full chain resolves empty.
- `project-env-var-still-tested` — `tests/test_pg_dispatch.py` sets `REASONS_PROJECT_ID` at lines 68 and 76, so the legacy env var is still an active input.
- `precedence-has-no-tests` — No test matches patterns pairing `domain_id`/`project_id` or `REASONS_DOMAIN_ID`/`REASONS_PROJECT_ID`, so the declared order is unprotected against regressions.
