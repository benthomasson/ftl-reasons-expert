---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: check_integrity in reasons/api.py

# `check_integrity` — `reasons/api.py`

## 1. Purpose

`check_integrity` is the read-only *verifier* of the RMS Merkle-hash chain. Every node stores a `text_hash` (hash of its text content) and every justification stores a `content_hash` (hash of its structure — antecedents, outlist, label — which transitively commits to the hashes of its antecedents). This function re-computes those hashes and reports mismatches, exposing:

- **text mutations** — a node's current text no longer matches its stored `text_hash`.
- **chain mutations** — a justification's antecedent/outlist/label structure no longer matches its stored `content_hash` (i.e., someone rearranged dependencies, dropped an antecedent, or edited the outlist).
- **missing hashes** — nodes/justifications that were never hashed.

It exists so callers (CLI `reasons check-integrity`, review tooling) can detect silent corruption or tampering in the belief graph without the runtime needing write access.

## 2. Contract

- **Preconditions**: `db_path` points to (or is a creatable path for) a SQLite network. No PostgreSQL backend is supported — unlike most sibling functions, this one takes no `pg_conninfo` / `domain_id`.
- **Postconditions**: Returns a JSON-serializable dict with exactly three keys: `text_mutations` (list), `chain_mutations` (list), `missing_hashes` (int). The database file is not modified.
- **Invariant**: Runs under `_with_network(db_path, write=False)` — the context manager guarantees no `Storage.save()` and no `pubsub.publish()` on exit.

## 3. Parameters

- `db_path: str = DEFAULT_DB` — filesystem path to the SQLite database (`DEFAULT_DB == "reasons.db"`).
  - **Edge case:** If the file doesn't exist, `_with_network` sets `mtime = None`, falls into the `else` branch, and constructs `Storage(db_path).load()`. Whether `Storage` auto-creates an empty DB (vs. raising) determines the observable behavior. The function itself has no existence check.
  - **Not accepted**: a PostgreSQL connection string, a `Storage` object, or a `Network` object.

## 4. Return Value

```python
{
  "text_mutations": list[dict],   # findings with type="text_mutation"
  "chain_mutations": list[dict],  # findings with type="chain_mutation"
  "missing_hashes": int,
}
```

- The finding dicts are returned verbatim from `verify_all` — the wrapper does not reshape them, only filters by `f["type"]`.
- **Caller must handle**: an empty `text_mutations` / `chain_mutations` is the only signal of "clean". Any non-empty list is a positive detection of tampering or drift. `missing_hashes > 0` means integrity of those entities cannot be trusted until `backfill_hashes` is run — the checker doesn't distinguish "unhashed but pristine" from "unhashed".

## 5. Algorithm

1. **Lazy import** `verify_all` from `reasons.merkle` *inside the function body*. This defers the import (avoids a module cycle: `merkle` presumably imports from `network`/`storage`) and keeps `reasons.api` import-time cheap.
2. Open the network with `_with_network(db_path, write=False)`. Because `write=False`:
   - If the file's `mtime` matches `_network_cache[db_path]`, the *cached* `Network` object is reused without touching disk.
   - Otherwise a fresh `Storage(db_path)` is loaded and, on `__exit__`, the cache entry is refreshed.
3. Call `verify_all(net)` — returns a dict with `findings` (list of dicts, each carrying at least a `type` key) and `missing_hashes` (int).
4. Partition `findings` with two list comprehensions: keep entries of type `"text_mutation"` and `"chain_mutation"`.
5. Discard everything else (any other finding types are **silently dropped** — see Error Handling below).
6. Build and return the trimmed dict.

## 6. Side Effects

- **Reads** the SQLite file (unless cache-hit).
- **May create** the SQLite file if `db_path` doesn't exist, depending on `Storage()`'s constructor behavior.
- **May update** `API._network_cache[db_path]` with the current `(mtime, Network)` tuple.
- **No writes to the belief network** — no `save()`, no pubsub events.
- **No logging** on its own; the network loader may log.

## 7. Error Handling

- **No try/except.** Anything raised by `verify_all`, `_with_network`, or `Storage` propagates to the caller.
- Possible exceptions:
  - `ImportError` if `reasons.merkle` is unavailable (lazy import).
  - `sqlite3.OperationalError` / `sqlite3.DatabaseError` if `db_path` is corrupt or locked.
  - `FileNotFoundError`/`PermissionError` if `Storage` doesn't auto-create and the path is bad.
  - `KeyError` if `verify_all` returns a dict without `"findings"` or `"missing_hashes"` — the wrapper assumes those keys exist.
  - `KeyError`/`TypeError` if an individual finding lacks `"type"` — the filter `f["type"]` will blow up rather than skipping.
- **Silent drop**: If `verify_all` ever grows a third `type` (e.g. `"orphan_mutation"`), this function will not surface it — the partitioned return dict is a *lossy* projection. This is a real API contract risk to review if `merkle` evolves.

## 8. Usage Patterns

Typical call sites (from context):

- CLI: `reasons check-integrity` → dumps the dict to stdout/JSON.
- Programmatic: a health check that asserts `not text_mutations and not chain_mutations and missing_hashes == 0`.
- Before trusting an imported external network (after `import_json` / `import_hf`) to see if anyone hand-edited the JSON.

**Caller obligations / gotchas**:

- Because of `_network_cache`, repeated calls within the same process on an unchanged file are cheap but can be **stale if the DB was modified by a process whose writes didn't update mtime** (e.g., WAL flush ordering); call `clear_network_cache(db_path)` if you need fresh verification.
- Do not interpret `missing_hashes > 0` as corruption — it just means hashes haven't been backfilled. Run `backfill_hashes` first if you need a meaningful integrity check.
- Only SQLite paths are supported; there is no `pg_conninfo` branch, unlike `check_stale`, `search`, etc.

## 9. Dependencies

| Kind | Symbol | Role |
|---|---|---|
| **Internal** | `reasons.merkle.verify_all` | Computes and compares hashes; produces `findings` + `missing_hashes`. |
| **Internal** | `reasons.api._with_network` | Read-only lifecycle manager; applies cache. |
| **Internal** | `reasons.storage.Storage` | Loads the network from SQLite (indirect via context manager). |
| **Internal** | `reasons.api.DEFAULT_DB` | Default path (`"reasons.db"`). |
| **stdlib** | `os.path.getmtime` | Cache-key computation inside `_with_network`. |
| **stdlib** | `sqlite3` | Via `Storage`. |

**Assumptions not enforced by the type system:**

1. `verify_all` returns a `dict` with keys `"findings"` (list of dicts with `"type"` ∈ strings) and `"missing_hashes"` (int). All three keys are looked up directly.
2. `db_path` is a filesystem path — passing a PG conninfo string would silently go to `sqlite3.connect` and fail confusingly.
3. The cached `Network` object in `_network_cache` was loaded from the *same* `db_path` (keyed by path, not by any DB identity).
4. `findings` entries never use `"type"` values other than the two reported — otherwise the wrapper's output is incomplete.

---

## Topics to Explore

- [function] `reasons/merkle.py:verify_all` — The core algorithm that walks nodes and justifications, recomputes `text_hash`/`content_hash`, and decides which findings (and which `type` strings) to emit — determines everything this wrapper reports.
- [function] `reasons/api.py:backfill_hashes` — The write counterpart: populates hashes that show up as `missing_hashes`, and is the prerequisite for a meaningful `check_integrity` run.
- [function] `reasons/api.py:_with_network` — Understanding the mtime-keyed cache (and the "read-only still constructs `Storage`" nuance) is essential to predicting `check_integrity`'s side effects and cross-process freshness.
- [file] `reasons/storage.py` — Whether `Storage(db_path)` on a missing path auto-creates an empty SQLite DB or raises, which determines `check_integrity`'s behavior on a nonexistent `db_path`.
- [general] `merkle-chain-design` — How `text_hash` and `content_hash` chain across a node's justifications transitively commits to antecedent hashes, which is what makes `chain_mutation` detection possible.

---

## Beliefs

- `check-integrity-requires-merkle-module` — `check_integrity` imports `reasons.merkle.verify_all` lazily inside its body, so a missing `merkle` module surfaces as `ImportError` only when the function is called.
- `check-integrity-is-read-only` — The function opens the network with `_with_network(db_path, write=False)`, so it never calls `Storage.save()` and never publishes pubsub events.
- `check-integrity-filters-finding-types` — Only findings whose `type` is `"text_mutation"` or `"chain_mutation"` are returned; any other finding type from `verify_all` is silently dropped.
- `check-integrity-reuses-network-cache` — A matching `(db_path, mtime)` in `_network_cache` causes the function to verify the cached `Network` without re-reading the SQLite file.
- `check-integrity-unsupported-for-postgres` — Unlike most sibling API functions, `check_integrity` takes no `pg_conninfo`/`domain_id` and only supports the SQLite backend.
