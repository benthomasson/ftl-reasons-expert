---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Nothing in this file checks invalid inputs, missing files, or DB failures; look for or add tests covering those, and inspect `sync_agent`'s exception handling.

# Analysis: Input Validation, Missing Files, DB Failures, and `sync_agent` Exception Handling

## What "this file" means in context

The code under scrutiny is **`reasons/import_agent.py`**, which contains both `sync_agent` (line 564) and `sync_agent_json` (line 585). The full source is available in the observations. It is a *pure in-memory transformation module*: it takes a `Network` object plus already-loaded text/dict payloads, mutates `Network.nodes`/`Network.nogoods`, and returns a summary dict. It never opens files, connects to a database, or parses its own inputs beyond what `parse_beliefs` and `_normalize_json` do.

So the premise "nothing in this file checks invalid inputs, missing files, or DB failures" is **literally true of `reasons/import_agent.py`** — but that is arguably by design, because those concerns belong to the boundary layers:

| Concern | Where it should live (and does) | Evidence |
|---|---|---|
| File existence / read errors | `reasons/api.py:2016` `sync_agent` | It dispatches to PG (`:2059`) or loads DB then calls `_sync_agent` (`:2078`–`:2094`); the actual file open is in this layer (not shown). |
| Invalid markdown / JSON | `parse_beliefs` (via `_normalize_markdown`) and `_normalize_json` | `reasons/import_agent.py:110` and `:141` |
| DB failures / transactions | `reasons/pg.py:2662` `PgApi.sync_agent` and `reasons/api.py` `_pg_dispatch` (`:2059`) | PG path off-loads to `PgApi`; SQLite path uses `Storage` |

The gap that actually matters is **`sync_agent`'s (and `sync_agent_json`'s) exception handling in `reasons/import_agent.py`** — that layer catches *nothing*.

---

## `sync_agent`'s exception handling (walkthrough)

`reasons/import_agent.py:564`:

```python
def sync_agent(network, agent_name, beliefs_text, nogoods_text=None, only_in=False, source_path=""):
    claims = _normalize_markdown(beliefs_text, only_in)
    nogoods = _normalize_nogoods_markdown(nogoods_text)
    return _sync_claims(network, agent_name, claims, source_path, nogoods)
```

Observations:

1. **No try/except anywhere on this path.** `_normalize_markdown` → `parse_beliefs` can raise on malformed markdown; `_normalize_nogoods_markdown` → `parse_nogoods` can raise on malformed nogoods. Those propagate raw to the caller.
2. **`agent_name` is never validated.** If it is `None`, `_ensure_agent_nodes` will f-string `None:active` (`import_agent.py:56`), which "works" but silently corrupts IDs. If it contains `:`, prefix-matching in `_sync_claims` (`import_agent.py:255`–`:258`) becomes ambiguous.
3. **`network` is assumed non-`None` and already constructed.** A `None` or wrong-type `network` raises `AttributeError` inside `_sync_claims`.
4. **`_sync_claims` is not atomic.** It mutates `network.nodes`, `network.nogoods`, and `network.repos` in place across ~200 lines (`import_agent.py:249`–`:471`). If it raises partway (e.g., a `KeyError` from `network.nodes[node_id]` in the removal loop near `:437`, or an `add_node` validation error), the caller's `Network` is **left partially mutated**. The outer layer's `Storage.save` may roll back the SQLite write, but the in-memory object is already dirty — a fact that any test for exception behavior must pin down.
5. **`sync_agent_json` (`:585`) is worse.** `_normalize_json` starts with `data.get("nodes", {})` (`:141`). If `data` is not a dict (e.g. `None`, a list), this raises `AttributeError` — no friendly `ValueError`. If `data["nodes"]` is a list, `nodes.items()` raises `AttributeError`. Same for `_normalize_nogoods_json(data)` calling `data.get("nogoods", [])` (`:193`).

   ```python
   # reasons/import_agent.py:585
   def sync_agent_json(network, agent_name, data, only_in=False, source_path=""):
       claims = _normalize_json(data, only_in)      # raises AttributeError on non-dict
       nogoods = _normalize_nogoods_json(data)      # same
       return _sync_claims(network, agent_name, claims, source_path, nogoods)
   ```

6. **No missing-file check, because no file is opened here.** `source_path` is only used to set `network.repos[agent_name] = str(Path(source_path).resolve().parent)` (`:266`–`:267`). `Path.resolve()` does *not* require the path to exist, so a bogus `source_path` silently records a nonsensical repo. Missing-file handling must live in `reasons/api.py:2016` and is already covered at the CLI level by `tests/test_cli.py:1202` `test_sync_agent_file_not_found`.

---

## Tests: what exists vs what's missing

### Covered already

- **Missing file (CLI level):** `tests/test_cli.py:1202 test_sync_agent_file_not_found`.
- **Happy-path sync semantics:** `tests/test_sync_agent.py` (636 lines) exercises add / remove / update text / dependents / truth-value flips / JSON sync / idempotency / cascade-on-revocation. Every test writes its input file to `tmp_path` first (`tests/test_sync_agent.py:47`, `:105`, `:169`, etc.), i.e. no error cases.
- **PG dispatch routing:** `tests/test_pg_dispatch.py:318–331` — but it uses `mock_pg.sync_agent.return_value = {...}`, so a *real* PG failure is never asserted.
- **Agent-level semantics:** `tests/test_import_agent.py` (499 lines) — same profile (all happy paths, files pre-written).

### Not covered (the real gaps)

1. **`api.sync_agent` with a non-existent file path at the *API* level** (as opposed to CLI). None of the 20+ calls in `tests/test_sync_agent.py` omits `p.write_text(...)`.
2. **Malformed markdown input:** e.g. `"###\n## random"`, unterminated fences, invalid `[IN/OUT]` tokens — dependent only on `parse_beliefs`, untested through `api.sync_agent`.
3. **Malformed JSON:** `sync_agent_json(None, ...)`, `sync_agent_json("not-a-dict", ...)`, `{"nodes": []}`, missing `"nodes"` key. None of the JSON tests in `tests/test_sync_agent.py:275–350` cover these.
4. **Invalid `agent_name`:** empty string, `None`, `"con:tained:colon"`. No test.
5. **DB failure during sync:** point `db_path` at a directory, a read-only file, or monkeypatch `Storage.save` to raise. No test. `tests/conftest.py`'s `pg_api` fixture (line ~25) skips when `DATABASE_URL` is unset, so the only "DB" test today is the mock in `tests/test_pg_dispatch.py:318`.
6. **Partial-mutation-on-exception:** no test asserts whether the in-memory `Network` survives an exception inside `_sync_claims` unchanged. This is the most important missing test for the exception-handling inspection.
7. **Exception contract at the API boundary:** is `api.sync_agent` supposed to raise, or return `{"error": ...}`? No test pins this down.

---

## Recommended tests to add

Add to **`tests/test_sync_agent.py`**:

```python
def test_sync_agent_missing_file_raises(db, tmp_path):
    with pytest.raises(FileNotFoundError):
        api.sync_agent("missing-agent", str(tmp_path / "nope.md"), db_path=db)

def test_sync_agent_invalid_markdown_raises(db, tmp_path):
    p = tmp_path / "beliefs.md"; p.write_text("not valid ### ## ###")
    with pytest.raises(Exception) as exc:
        api.sync_agent("bad-md", str(p), db_path=db)
    # pin the actual exception type here (from parse_beliefs)

def test_sync_agent_json_non_dict_payload(db):
    with pytest.raises(AttributeError):   # or convert to ValueError in source
        api.sync_agent_json("bad-json", None, db_path=db)

def test_sync_agent_empty_agent_name_rejected(db, tmp_path):
    p = tmp_path / "b.md"; p.write_text(SAMPLE)
    with pytest.raises((ValueError, KeyError)):
        api.sync_agent("", str(p), db_path=db)

def test_sync_agent_db_failure_does_not_corrupt_network(db, tmp_path, monkeypatch):
    # monkeypatch Storage.save to raise; assert no partial state was persisted
    ...
```

Add to **`tests/test_pg_dispatch.py`**: a variant where `mock_pg.sync_agent.side_effect = RuntimeError("boom")` and assert the exception surfaces (or is wrapped) — testing that DB errors from `PgApi.sync_agent` (`reasons/pg.py:2662`) are not silently swallowed.

Add to **`tests/test_pg.py`** (not shown): a real-PG test using the `pg_api` fixture that drops the connection mid-sync and asserts the transaction rolls back, since `tests/conftest.py:29` already wires the cleanup.

---

## Caveats on the observations

- I do **not** have the body of `reasons/api.py::sync_agent` (lines 2016–2094) or `reasons/pg.py::sync_agent` (line 2662). Whether `api.sync_agent` wraps `_sync_agent` in a transaction, catches exceptions, or converts them to error dicts is **unverified** from these observations. The recommendation to pin those contracts down is the reason to add tests that don't rely on those sources.
- The rest of `tests/test_sync_agent.py` (lines 501–636) is not visible; it *might* contain a `test_sync_agent_file_not_found`-style test at the API level, but the excerpt exposes only lines 0–500 and every visible test pre-writes its file. That is the assumption I'm operating under.

## Topics to Explore

- [function] `reasons/api.py:sync_agent` — The boundary layer where file reads, DB open, and PG dispatch actually happen; confirm whether it wraps `_sync_agent` in a transaction and how it surfaces `FileNotFoundError` / DB errors.
- [function] `reasons/pg.py:sync_agent` — The PG implementation at line 2662; inspect whether it commits/rolls back atomically and what exception types it lets escape.
- [file] `reasons/import_beliefs.py` — `parse_beliefs` / `parse_nogoods` are the only validation layer for markdown; determine what inputs they reject vs silently coerce.
- [file] `tests/test_cli.py` — `test_sync_agent_file_not_found` (line 1202) proves missing-file handling exists at the CLI; verify whether the same guard exists at the API level.
- [general] `partial-mutation-on-exception` — Because `_sync_claims` mutates the in-memory `Network` in place with no try/except, any raised exception leaves the caller's object dirty; explore whether callers reload from DB or reuse the object.

## Beliefs

- `sync-agent-does-not-open-files` — `sync_agent` in `reasons/import_agent.py:564` takes `beliefs_text` as a parameter and never opens files; file I/O is the responsibility of `reasons/api.py:sync_agent`.
- `sync-claims-is-not-atomic` — `_sync_claims` (reasons/import_agent.py:249) mutates `network.nodes`, `network.nogoods`, and `network.repos` in place with no try/except and no rollback.
- `sync-agent-json-raises-attribute-error-on-non-dict` — `sync_agent_json` (reasons/import_agent.py:585) calls `_normalize_json`, which uses `data.get(...)` and will raise `AttributeError` if `data` is not a dict.
- `missing-file-covered-only-at-cli` — `tests/test_cli.py:1202 test_sync_agent_file_not_found` is the only visible test for missing files; no test in `tests/test_sync_agent.py` exercises a nonexistent path.
- `pg-dispatch-tests-use-mocks` — `tests/test_pg_dispatch.py:318` mocks `PgApi.sync_agent`, so real PG failures are not exercised by the existing suite.
- `sync-agent-has-no-input-validation` — `reasons/import_agent.py:sync_agent` never validates `agent_name`, `network`, or `beliefs_text` before delegating to `_normalize_markdown` / `_sync_claims`.
