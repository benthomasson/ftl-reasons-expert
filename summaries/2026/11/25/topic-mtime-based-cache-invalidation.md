---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore edge cases where file mtime is insufficient to detect external database changes.

# Edge Cases Where File mtime Is Insufficient

## The mechanism being questioned

In `reasons/api.py`, the network cache uses the database file's modification time as its sole invalidation signal:

- `reasons/api.py:127` — reads `mtime = os.path.getmtime(db_path)`
- `reasons/api.py:131` — serves the cached `Network` when `cached[0] == mtime and not write`
- `reasons/api.py:163–164` — recomputes `mtime` after building the network, then stores `_network_cache[db_path] = (mtime, self.network)`

The related test — `tests/test_network_cache.py:83` — asserts the naive model: *"If the db file is modified externally, mtime changes and cache is bypassed."* That assumption is where the edge cases live.

A key architectural detail complicates this immediately: `Storage.__init__` in `reasons/storage.py` runs `PRAGMA journal_mode=WAL` (`reasons/storage.py`, `Storage.__init__`). In WAL mode, committed writes are appended to a `<db_path>-wal` sidecar file and are only merged into the primary `.db` file at checkpoint time (default ~1000 pages, or on clean close). So the file `os.path.getmtime` inspects in `reasons/api.py:127` / `:163` is **not** the file that receives most external writes.

## Concrete failure modes

1. **WAL sidecar writes don't bump the main file's mtime.**
   An external process (a second `Storage` instance, `sqlite3` CLI, a migration script) commits a transaction. SQLite appends the changed pages to `reasons.db-wal`. `os.path.getmtime("reasons.db")` in `reasons/api.py:127` is unchanged until a checkpoint fires. The cached `Network` at `reasons/api.py:131` is returned stale. This is the highest-probability edge case because WAL is the default used by `Storage` itself.

2. **Timestamp-granularity collisions.**
   `reasons/api.py:131` compares floats with `==`. Many filesystems (older ext3, HFS+, FAT32, SMB/CIFS mounts, some NFS variants) round to 1s or 2s. An external writer that commits within the same tick produces the same `st_mtime` value as the prior cache entry, so `cached[0] == mtime` is `True`. This is especially likely for the *first* external write after a read.

3. **mtime deliberately preserved by tooling.**
   `rsync -a`, `tar -p`, `cp -p`, `git checkout`, and any script doing `os.utime(path, (old_atime, old_mtime))` can replace the database content while restoring the original timestamp. Restoring an expert database from a backup snapshot (or a replica arriving over a snapshotting transport) is a realistic case where the file lives at the same path, has different content, and keeps its mtime.

4. **Atomic replace via temp-file rename.**
   Common pattern: write new `db.tmp`, `os.replace("db.tmp", "db")`. If the temp file was staged minutes ago (e.g., a rotated backup) or written within the same coarse-timestamp tick, the newly visible `reasons.db` has the old mtime the cache recorded. Content differs, mtime matches, cache serves stale data.

5. **Race between load and re-read at `reasons/api.py:163`.**
   The function reads the network, then re-reads `getmtime` afterwards and stores `(new_mtime, self.network)`. If an external writer commits *between* the load and line 163, the cached tuple records a post-change mtime paired with a pre-change `Network`. Every subsequent call at line 131 sees a matching mtime and returns data that never reflected the external change. This is an mtime-only invalidation scheme failing even when mtime *is* changing.

6. **Attribute caching on network filesystems.**
   On NFS/SMB, `os.path.getmtime` may return a client-cached attribute for several seconds after a remote write. Even if the file is local to the DB server, a client on the same mount can miss the change entirely for the attribute-cache lifetime.

7. **Clock skew across multiple writers.**
   If an external writer is on a machine whose clock is behind the reader's, the new mtime can be equal to or less than the cached mtime. A backwards jump triggers a miss (`!=` compares unequal) so it's safe but wasteful; the dangerous case is an *equal-second* clock collision, which repeats (2) but across hosts and cannot be closed by increasing local timestamp resolution.

8. **Multiple symlinked paths to the same inode.**
   The cache is keyed by `db_path` (`reasons/api.py:131`, `:164`), and only the mtime for that path is stored. Two calls (`/data/reasons.db` vs `/data/link-to-reasons.db`) hit the same inode with the same `getmtime`, but each caches its own tuple. An external write invalidates both simultaneously given (1)–(7) resolve, but a caller that reads via a different path can be holding a stale entry from before the symlink was retargeted. Minor, but relevant if the symlink is used to switch between expert repos.

9. **Long-lived connection vs file level.**
   Even when `reasons/api.py` reloads the `Network` because `mtime` changed, the reload goes through a `Storage` connection established in `Storage.__init__`. SQLite's cross-connection visibility of WAL data depends on the WAL index; this is normally correct, but a process that has been idle and whose page cache references pages superseded by a checkpoint race can still momentarily observe data older than the visible `mtime` suggests. mtime is a signal about the *file*, not about consistency of the process's view.

10. **Case-insensitive or path-normalized filesystems.**
    `_network_cache` is a plain `dict` keyed by the string `Path(db_path)`. On macOS with APFS (case-insensitive by default), `/data/Reasons.db` and `/data/reasons.db` resolve to the same inode but are cached as two entries with independent tuples. One entry can be invalidated while the other stays stale relative to the same underlying data.

## What the observations don't show

The grep results for `reasons/api.py` only surface isolated lines (119, 127–131, 163–164) — the enclosing function, exception handling, whether `_network_cache` is a process-local dict or shared, and whether a checkpoint is triggered on write paths are **not** visible. The full bodies of `reasons/api.py` and the write path in `Storage` (specifically whether `Storage` runs `PRAGMA wal_checkpoint` after writes) would close the gap. `reasons/storage.py` was only shown up to line 200 and cuts off in `_init_schema`.

Also note that the tests in `tests/test_network_cache.py` (per the visible snippet, lines 83–99) verify *mtime changes → cache miss* using normal `write_text` calls; they do **not** cover WAL sidecar-only writes, `os.utime` preservation, second-resolution filesystems, or the load/re-read race at `reasons/api.py:163`.

## Topics to Explore

- [file] `reasons/api.py` — The full function containing the cache logic (around lines 119–164); confirm exception handling, whether `_network_cache` is process-local, and whether the load runs against a fresh or long-lived connection.
- [function] `reasons/storage.py:Storage.__init__` — Where `PRAGMA journal_mode=WAL` is configured; inspect whether any path runs `wal_checkpoint`, since that determines how visible external writes are to `getmtime("reasons.db")`.
- [file] `tests/test_network_cache.py` — Existing invalidation tests; extend with `os.utime`, second-resolution, and concurrent-writer scenarios.
- [general] `sqlite-wal-checkpoint-visibility` — Research how long external WAL commits remain invisible to the main file's mtime and how to force a checkpoint with `PRAGMA wal_checkpoint(TRUNCATE)`.
- [general] `mtime-bypass-strategies` — Alternatives worth considering: hashing the DB (as `reasons/check_stale.py:hash_file` already does for source files), reading `PRAGMA data_version` / `PRAGMA schema_version`, or listening to SQLite update hooks.

## Beliefs

- `network-cache-keyed-by-db-path-and-mtime` — The cache in `reasons/api.py` is keyed by `(db_path, os.path.getmtime(db_path))` and is only bypassed when `write=True` or when the stored mtime differs (line 131).
- `storage-enables-wal` — `Storage.__init__` in `reasons/storage.py` executes `PRAGMA journal_mode=WAL`, so writes made by external processes reside in a `-wal` sidecar until a checkpoint moves them into the main file.
- `mtime-reread-creates-stale-cache-window` — `reasons/api.py:163` re-reads `getmtime` *after* the network is loaded and stores it with the loaded network, so an external commit between load and re-read produces a cache entry whose mtime postdates its content.
- `no-st-mtime-ns-usage` — The `find_stat_mtime_usage` scan returns zero matches, so no code path reads `st_mtime_ns`; all invalidation relies on the lower-precision `os.path.getmtime` float.
- `write-calls-always-bypass-cache` — Line 131 gates the fast path on `not write`, so writes made through the API itself are never served from the cached `Network`, regardless of mtime.
