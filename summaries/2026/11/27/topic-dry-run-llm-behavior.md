---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Determine whether dry-run prevents only database writes or also avoids model calls.

# Does `dry_run` prevent only DB writes, or also model calls?

**Short answer:** It depends on the function. `dry_run` is not a single, global semantic — each API/CLI function defines its own meaning. The codebase shows **two distinct interpretations**:

1. **Write-guard only** (model calls still happen) — used by the classification path.
2. **Skip the expensive operation entirely** (prompt is built, model is *not* invoked) — used by the derivation path.

---

## The write-guard mechanism (shared foundation)

The `_with_network` context manager at `reasons/api.py:111` implements the "no DB write" behavior:

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    if write and exc_type is None:
        if self.store:
            self.store.save(self.network)   # <-- only called when write=True
```

Any API function that calls `_with_network(db_path, write=not dry_run)` therefore guarantees that when `dry_run=True`, the network is loaded but **never saved**. The clearest instance is at `reasons/api.py:1562`:

```python
with _with_network(db_path, write=not dry_run) as net:
```

That guards the `migrate_retract_to_defeaters` path. It has no model calls of its own, so `dry_run` there is pure "don't persist."

---

## Case 1 — `classify_defeat_reason_types`: **model calls DO happen**

The clearest evidence is the docstring at `reasons/api.py:1669`:

> `dry_run: If True, classify but don't write to db`

That is unambiguous: classification (which requires an LLM — see the `model=args.model` parameter at `reasons/cli.py:296` and `reasons/llm.py:invoke_model`) still runs. Only the persistence step is skipped, gated by `if not dry_run:` at `reasons/api.py:1714`.

The CLI wrapper at `reasons/cli.py:294-298` confirms this — `model` is passed through regardless of `dry_run`:

```python
dry_run = not args.apply
result = api.classify_defeat_reason_types(
    ...
    model=args.model,
    dry_run=dry_run,
    ...
)
if dry_run:
    print("Dry run — no changes applied")
```

So a user running `reasons classify-defeaters` without `--apply` still pays the token cost. That is the intended contract.

---

## Case 2 — `derive`: **model call is skipped**

The CLI test `tests/test_cli.py::TestDerive::test_derive_dry_run` asserts that `derive --dry-run` succeeds and the output contains `"Prompt"` and the belief id `"a"`. That strongly suggests the command *builds the prompt via `reasons/derive.py:build_prompt`* and prints it — but never dispatches it to `invoke_model`. `build_prompt` is pure Python (string aggregation over the network; see `reasons/derive.py:build_prompt`). So here `dry_run` prevents both the DB write *and* the LLM invocation.

The empty-network test `test_derive_dry_run_empty` also exits with `code == 1`, i.e., "nothing to prompt for" — again consistent with "build and show the prompt, don't call the model."

*(Caveat: the CLI handler for `derive` is past line 1200 of `reasons/cli.py`, so the exact branch `if dry_run: print(prompt)` isn't visible in the supplied observations — it is inferred from the test. The api-level `dry_run: bool = False` signatures at `reasons/api.py:4625`, `4689`, and their internal `dry_run=True, db_path=db_path` calls at `4646`, `4710` look like the derive/propose pipeline, but the bodies aren't shown.)*

---

## Case 3 — `migrate_retract_to_defeaters`: **no model, only write-guard**

CLI: `reasons/cli.py:291` sets `dry_run = not args.apply` and passes it straight to `api.migrate_retract_to_defeaters` (`cli.py:294`). The API wraps the work in `_with_network(db_path, write=not dry_run)` at `api.py:1562` and gates the actual migration with `if not dry_run and migrated:` at `cli.py` output / `api.py:1646`. No model is involved in this path at all — so `dry_run` never had a chance to suppress one.

---

## Takeaway

| Function | `dry_run=True` prevents DB write? | `dry_run=True` prevents model call? |
|---|---|---|
| `migrate_retract_to_defeaters` (`api.py:1551`) | ✅ (`write=not dry_run`) | n/a — no model |
| `classify_defeat_reason_types` (`api.py:1656`, doc `:1669`) | ✅ (`if not dry_run:` `:1714`) | ❌ — **classify anyway** |
| `derive` (CLI `--dry-run`) | ✅ | ✅ (test asserts only `"Prompt"` printed) |

So there is **no single answer** — read the docstring per function. The `classify` docstring (`"classify but don't write to db"`) is the canonical example of "dry-run does NOT skip the model"; the `derive` CLI test is the canonical example of "dry-run builds the prompt but does not invoke the model."

---

## Topics to Explore

- [function] `reasons/api.py:classify_defeat_reason_types` — Read the full body (lines ~1656–1720) to see exactly where the LLM is called and where the result dict is discarded under `dry_run`.
- [file] `reasons/cli.py` — Focus on the `derive` command handler (past line 1200) to confirm the `if dry_run: print(prompt)` behavior inferred from `tests/test_cli.py::TestDerive`.
- [function] `reasons/api.py:_with_network` — The `__exit__` at ~line 137 is the single choke point that makes "no write" trustworthy for every `dry_run` path.
- [file] `reasons/llm.py` — `invoke_model` and `_record_cost`; confirms that whenever an `invoke_model` call fires, tokens/cost are accrued even if the caller later drops the result (relevant for `classify-defeaters --dry-run`).
- [general] `dry-run-semantics-audit` — Grep every `dry_run` signature in `reasons/api.py` and check whether its docstring says "classify/preview but don't write" (model runs) vs. "show what would happen" (model skipped).

---

## Beliefs

- `dry-run-varies-by-function` — The meaning of `dry_run` is function-specific; it is not a single global "no model" switch.
- `classify-dry-run-still-calls-model` — `classify_defeat_reason_types(dry_run=True)` invokes the LLM and only skips the DB write, per the docstring at `reasons/api.py:1669`.
- `with-network-write-flag-enforces-no-write` — `_with_network(db_path, write=not dry_run)` guarantees no `Storage.save()` occurs when `dry_run=True` (via `__exit__` at `reasons/api.py:~137`).
- `derive-dry-run-skips-model-call` — `derive --dry-run` builds the prompt via `derive.build_prompt` and prints it without calling `invoke_model` (evidenced by `tests/test_cli.py::TestDerive::test_derive_dry_run`).
- `migrate-has-no-model-dependency` — `migrate_retract_to_defeaters` performs no LLM call at all, so `dry_run` there is purely a write-guard.
