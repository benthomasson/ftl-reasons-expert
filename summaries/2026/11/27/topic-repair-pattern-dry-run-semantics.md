---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Why `dry_run` returns `"linked"` even though no mutation occurred, and where the caller is expected to reconcile this.

## Why `dry_run` still returns `"linked"`

The `"linked"` status is **prospective**, not a statement of fact. It is produced during the classification/match phase, before any write takes place, and `dry_run` is only consulted later at the mutation gate.

Concretely, the two responsibility sites in `reasons/repair.py` are:

- **`reasons/repair.py:357`** — the classifier returns `"linked"` as one of its status outcomes, with no reference to `dry_run`:
  ```python
  return "linked", claim, matched_ids, match.get("rationale", "")
  ```
- **`reasons/repair.py:437`** — the only place where that status is converted into a mutation, gated on `dry_run`:
  ```python
  if not dry_run and status == "linked":
      ...
  ```

So the return value answers *"what would happen if we applied this?"*, not *"what did we do?"*. When `dry_run=True`, the classifier still reports `"linked"`, but the write at line 437 is skipped — hence "linked" with no mutation.

The same convention appears in the higher-level API wrappers:

- `reasons/api.py:3853-3860` — a per-result dict is annotated with `"dry_run": True` in the dry-run path.
- `reasons/api.py:3905` — the applied path annotates `"dry_run": False`.
- `reasons/api.py:4710` / `reasons/api.py:4735` — the same worker call is made once with `dry_run=True` (preview) and once with the caller-supplied `dry_run`.

## Where the caller is expected to reconcile

There is **no named `reconcile` symbol** in the codebase (`find_reconcile_references` returns `0 matches`). Reconciliation is a convention rather than an explicit function. It happens in two layers:

**1. The API summarizers deliberately suppress the `"linked"` count when `dry_run=True`.**

- `reasons/api.py:4722` — dry-run summary hard-codes `"linked": 0,` even though the worker returned statuses of `"linked"`.
- `reasons/api.py:4741` — the applied summary sums real statuses:
  ```python
  "linked": sum(1 for r in results if r["status"] == "linked"),
  ```

That asymmetry (`0` vs. `sum(...)`) is the reconciliation: the summary layer is where "would-be linked" gets erased so downstream consumers reading the counters see zero applied links under dry-run.

**2. The CLI branches on `dry_run` after the call.**

- `reasons/cli.py:291` — `dry_run = not args.apply`
- `reasons/cli.py:294` — `node_ids=node_ids, dry_run=dry_run, db_path=args.db,`
- `reasons/cli.py:297` — `if dry_run:` gate for presenting the preview vs. the applied result.

**Implication for direct callers:** anyone calling the repair worker below `api.py` — or reading `status` from a result dict that also carries `"dry_run": True` (`api.py:3860`) — is responsible for treating `"linked"` as "would link". The API and CLI honor this; nothing structurally prevents a new caller from ignoring it and miscounting dry-run statuses as real links.

## What the observations do not show

The full bodies of `reasons/repair.py` around lines 300–450 and `reasons/api.py` around 4625–4741 are not in the observations. The exact function names of the wrappers and the exact shape of the return tuple from `repair.py:357` cannot be confirmed from the snippets alone; the reasoning above is inferred from the call sites and the `"linked": 0` vs. `sum(...)` asymmetry. A repository-level contract (type annotation, docstring, or JSON schema) stating that `status == "linked"` is prospective under `dry_run` is likewise not observed.

## Topics to Explore

- [file] `reasons/repair.py` — Read the full body around lines 300–450 to see how `status` is computed independently of `dry_run` and what else the tuple at line 357 carries.
- [file] `reasons/api.py` — Inspect the wrappers at ~4689–4741 to confirm how the dry-run summary (`"linked": 0`) is produced vs. the applied summary (`"linked": sum(...)`).
- [file] `reasons/cli.py` — Check the dry-run branch at lines 291–297 to see how the preview vs. applied distinction is presented to users.
- [general] `dry-run-contract` — Whether a documented/typed contract exists that `status == "linked"` is prospective, and whether `"dry_run": True` in a per-result dict is the authoritative signal callers must check before counting mutations.
- [general] `repair-status-taxonomy` — Whether `"softened"` and `"abandoned"` share the same prospective-vs-applied semantics as `"linked"`, so all three counters need the same dry-run suppression.

## Beliefs

- `dry-run-does-not-gate-status-classification` — The classifier at `reasons/repair.py:357` returns `"linked"` without consulting `dry_run`; the flag only gates the write at `reasons/repair.py:437`.
- `dry-run-summary-zeroes-linked-count` — The dry-run summary in `reasons/api.py:4722` hard-codes `"linked": 0`, while the applied summary in `reasons/api.py:4741` uses `sum(1 for r in results if r["status"] == "linked")`.
- `per-result-dicts-carry-dry-run-flag` — Result dicts in `reasons/api.py` include `"dry_run": True` (line 3860) or `"dry_run": False` (line 3905), making the prospective-vs-applied distinction available to downstream callers.
- `no-explicit-reconcile-symbol-exists` — A codebase-wide grep for `"reconcile"` yields zero matches; dry-run reconciliation is convention, not a named function.
- `cli-branches-on-dry-run` — The CLI derives `dry_run = not args.apply` at `reasons/cli.py:291` and has a dedicated `if dry_run:` branch at `reasons/cli.py:297` for presenting results.
