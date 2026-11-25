---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Design a round-trip test that runs `pin_sources()` → `Storage.save()` → `Storage.load()` and asserts `node.metadata["pinned_sha"]`, plus an equivalent two-`PgApi`-connection test for `verified_at`.

## Goal

The SQLite test should exercise the real persistence path:

1. `pin_sources()` mutates an in-memory `Network` and writes `node.metadata["pinned_sha"]`.
2. `Storage.save(net)` writes that network to SQLite.
3. A fresh `Storage(...).load()` reconstructs a new `Network`.
4. Assert the reconstructed node still has the pinned SHA.

The PostgreSQL analogue should use two separate `PgApi` instances so the assertion proves the value was read back from PostgreSQL, not from an in-memory object.

Relevant code paths from the observations:

- `pin_sources()` in `reasons/check_stale.py:311`; it writes `node.metadata["pinned_sha"]` at `reasons/check_stale.py:349` and `node.metadata["verified_at"]` at `reasons/check_stale.py:350`.
- Existing pin tests already exercise `pin_sources()` in `tests/test_pin_sources.py::TestPinSources`.
- `Storage` is defined at `reasons/storage.py:123`.
- Existing round-trip tests use `Storage.load()` through `_load_network()` in `tests/test_roundtrip.py`, and `Storage.save(net)` is used in `tests/test_roundtrip.py::TestFullRoundTrip::test_all_fields_survive`.
- `PgApi` is defined at `reasons/pg.py:98`.
- `PgApi` reads the first-class `verified_at` column and maps it to `"verified_at"` at `reasons/pg.py:732`.

## SQLite round-trip test for `pinned_sha`

Add this to `tests/test_pin_sources.py`, alongside `TestPinSources`. You will need to import `api` and `Storage`:

```python
from reasons import api
from reasons.storage import Storage
```

Test:

```python
def test_pinned_sha_survives_storage_round_trip(self, git_repo, tmp_path):
    db_path = str(tmp_path / "pin_roundtrip.db")
    api.init_db(db_path=db_path)

    net = Network()
    net.add_node(
        "belief-1",
        "Test belief",
        source="source.md",
        source_hash="",
    )

    results = pin_sources(net, db_dir=git_repo)
    assert len(results) == 1

    expected_sha = net.nodes["belief-1"].metadata["pinned_sha"]
    assert len(expected_sha) == 40
    assert results[0]["pinned_sha"] == expected_sha

    storage = Storage(db_path)
    storage.save(net)
    storage.close()

    storage = Storage(db_path)
    loaded = storage.load()
    storage.close()

    assert loaded.nodes["belief-1"].metadata["pinned_sha"] == expected_sha
```

Why this shape:

- `git_repo` is already the fixture in `tests/test_pin_sources.py` that creates a committed `source.md`, so `pin_sources(..., db_dir=git_repo)` can resolve the commit SHA.
- `api.init_db(db_path=db_path)` mirrors the `db` fixture in `tests/test_roundtrip.py`.
- `Storage.save(net)` followed by a fresh `Storage(db_path).load()` is the same pattern used by `_load_network()` in `tests/test_roundtrip.py`.
- The assertion on `loaded.nodes["belief-1"].metadata["pinned_sha"]` directly verifies the field written by `pin_sources()` at `reasons/check_stale.py:349`.

If this test fails, the likely issue is that `Storage.save()` / `Storage.load()` is not preserving `Node.metadata`, which is exactly what the test is meant to catch.

## PostgreSQL two-`PgApi`-connection test for `verified_at`

For PostgreSQL, the target is the first-class `Node.verified_at` field, not `node.metadata["verified_at"]`. The PG schema has a `verified_at TIMESTAMPTZ` column at `reasons/pg.py:36`, and `PgApi` maps it into the node representation at `reasons/pg.py:732`.

The intended test shape is:

```python
@pytest.mark.pg
def test_verified_at_visible_to_second_pg_connection(pg_api, pg_conninfo, pg_project):
    pg_api.add_node("pg-roundtrip", "Alpha")

    # Stamp verified_at through the real PgApi write path.
    pg_api.set_node_verified_at("pg-roundtrip", "2026-06-04T12:00:00Z")

    first_read = pg_api.show_node("pg-roundtrip")
    expected = first_read["verified_at"]
    assert expected

    api2 = PgApi(pg_conninfo, pg_project)
    try:
        second_read = api2.show_node("pg-roundtrip")
        assert second_read["verified_at"] == expected
    finally:
        api2.close()
```

The important property is that `api2` is a separate `PgApi` connection/instance, so the second read cannot be satisfied by a cached in-memory `Network`.

However, the observations are missing two pieces needed for an exact runnable PG test:

1. The `PgApi` constructor signature is not visible; only `class PgApi` at `reasons/pg.py:98`.
2. There is no observed public `PgApi` method that writes `verified_at`. The excerpt shows PG schema/read mappings for `verified_at` at `reasons/pg.py:36`, `reasons/pg.py:683-732`, and `reasons/pg.py:1348-1376`, but not the write path.

So in the real test, replace the placeholder:

```python
pg_api.set_node_verified_at(...)
```

with whichever production path stamps `verified_at`. If no public `PgApi` setter exists, the test should either:

- add/expose one, or
- use the same SQL update path used by production, through the test connection, then assert through a second `PgApi`.

Do not simply set `pg_api.show_node(...)["verified_at"]` in Python; that would not test PostgreSQL persistence.

Also note the distinction:

- `pin_sources()` writes `node.metadata["verified_at"]` at `reasons/check_stale.py:350`.
- `PgApi` exposes the first-class node column `verified_at` at `reasons/pg.py:732`.

Those are not automatically the same thing.

## Topics to Explore

- [file] `reasons/storage.py` — Inspect `Storage.save()` and `Storage.load()` to confirm how `Node.metadata` is serialized; the observation only gives the class line and schema fragments.
- [function] `reasons/check_stale.py:pin_sources` — Read the full function to confirm all metadata keys and conditions it writes.
- [file] `reasons/pg.py` — Inspect `PgApi` constructor, write methods, and the SQL around lines 683–732 and 1348–1376 to find the exact `verified_at` write path.
- [file] `tests/conftest.py` — Find the `pg_api` fixture definition and how to create a second `PgApi` with the same connection/project settings.
- [general] `metadata-vs-node-fields` — Understand why `node.metadata["verified_at"]` and `node.verified_at` are different persistence targets.

## Beliefs

- `pin-sources-writes-pinned-sha-to-metadata` — `pin_sources()` assigns `node.metadata["pinned_sha"]` in `reasons/check_stale.py:349`.
- `storage-load-pattern-exists-in-roundtrip-tests` — `tests/test_roundtrip.py` loads a network via `Storage(db_path).load()` and closes the storage handle.
- `pgapi-show-node-reads-verified-at-column` — `PgApi` maps the selected `verified_at` column into the returned node data at `reasons/pg.py:732`.
- `verified-at-is-first-class-node-field` — `Node.verified_at` exists independently of `node.metadata`, as shown by `reasons/__init__.py:48` and `reasons/network.py:84`.
- `storage-schema-has-verified-at-column` — The SQLite schema declares `verified_at TEXT DEFAULT ''` at `reasons/storage.py:32`.
