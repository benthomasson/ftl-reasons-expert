---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _find_db in reasons/mcp_server.py

`_find_db` is an internal, zero-argument path resolver for the MCP server. It decides which `reasons.db` path the server should use, with this precedence:

1. Non-empty `REASONS_DB` environment variable.
2. Nearest `reasons.db` found by walking upward from the current working directory.
3. `api.DEFAULT_DB`.

## Purpose

The MCP server needs a database path before it can execute belief-network tools. `_find_db` exists so that users can launch the server from a subdirectory of a project and still discover the project’s `reasons.db`, matching the CLI’s intended discovery behavior. It also supports an explicit override via `REASONS_DB` and provides a final fallback through `api.DEFAULT_DB`.

## Contract

### Preconditions

- The process must have a readable current working directory for the walking behavior. `os.getcwd()` can raise if the cwd is invalid or inaccessible.
- `api.DEFAULT_DB` must exist as a string-like path if no override or discovered file is found.
- If `REASONS_DB` is set to a non-empty value, it is assumed to be a usable path by the caller, but `_find_db` does not enforce that.

### Postconditions

- If `REASONS_DB` is set to a truthy string, the function returns that exact string immediately.
- Otherwise, it returns the first existing path equal to `<ancestor>/reasons.db`, starting from `os.getcwd()` and walking upward.
- If no such path is found, it returns `api.DEFAULT_DB`.
- The returned path is not guaranteed to exist in the `REASONS_DB` and fallback cases.
- The returned path is not guaranteed to be a valid SQLite database, or even a regular file, in the discovered-file case. `os.path.exists` returns true for directories too.

### Invariants

- The function does not change the current working directory.
- The function does not modify the environment.
- The function does not create, open, or validate the database.
- For discovered paths, the basename is exactly `"reasons.db"`.
- The function itself does not cache the result; caching is performed by `_get_db`.

### Assumptions not enforced by the type system

- `os.environ.get("REASONS_DB")` is treated as “not configured” when it is the empty string, because `""` is falsy.
- A value like `"~/reasons.db"` or `"$HOME/reasons.db"` is returned literally, without `expanduser` or `expandvars`.
- A relative `REASONS_DB` is returned as-is.
- `api.DEFAULT_DB` is assumed to be a string path; the annotation does not prove that.
- The function assumes “exists” is sufficient for the MCP server’s purposes, but it does not verify file type, permissions, schema, or readability.

## Parameters

None.

## Return Value

Returns a `str` path. There are three cases:

- `REASONS_DB` is truthy: returns the environment value verbatim.
- An ancestor contains `reasons.db`: returns the path formed by joining that ancestor directory and `"reasons.db"`.
- No environment override and no ancestor match: returns `api.DEFAULT_DB`.

The caller must be prepared for:

- a path that does not exist, especially via `REASONS_DB` or `api.DEFAULT_DB`;
- a relative path via `REASONS_DB`;
- a path to something that is not a valid SQLite database;
- a cached result, if the caller goes through `_get_db`.

## Algorithm

1. Check `os.environ.get("REASONS_DB")`.
2. If the resulting value is truthy, return it immediately.
3. Set `d = os.getcwd()`.
4. In an infinite loop:
   - Build `candidate = os.path.join(d, "reasons.db")`.
   - If `os.path.exists(candidate)`, return `candidate`.
   - Compute `parent = os.path.dirname(d)`.
   - If `parent == d`, break. This detects a filesystem root or drive root.
   - Otherwise, set `d = parent` and repeat.
5. After the loop breaks, return `api.DEFAULT_DB`.

Note that the loop checks the root directory itself before breaking. For example, starting anywhere on a POSIX system, it will eventually check `/reasons.db` before falling back.

## Side Effects

- Reads `REASONS_DB` from the process environment.
- Reads the current working directory via `os.getcwd()`.
- Performs filesystem existence checks via `os.path.exists`.
- No writes, no database access, no caching, and no global state changes inside `_find_db` itself.

## Error Handling

`_find_db` has no `try/except` block.

- `os.getcwd()` may raise `OSError` or a subclass if the current directory is invalid or inaccessible. That exception propagates.
- `os.path.exists` generally returns `False` for failed `stat` calls rather than propagating most filesystem errors, so permission problems may appear as “not found.”
- If `api.DEFAULT_DB` is missing, Python will raise `AttributeError`. That propagates.
- The function does not raise an error when `REASONS_DB` points to a missing file or invalid database. Returning the bad path is intentional at this layer; failure occurs later when the API tries to use it.

## Usage Patterns

`_find_db` is normally called indirectly:

- `main()` calls `_db = _find_db()` before starting `mcp.run()`.
- `_get_db()` lazily calls `_find_db()` the first time a tool needs the database, then caches the result in `_db`.

Because `_get_db` caches the path, later changes to the working directory, environment variable, or filesystem are not re-discovered after the first resolution. Also note that `main()` eagerly resolves the path before the server starts, so the server’s lifetime typically uses one path.

Callers should ensure that either:

- `REASONS_DB` points to a valid database,
- a valid `reasons.db` is discoverable from the startup directory, or
- `api.DEFAULT_DB` is acceptable for the environment.

## Dependencies

- `os` from the Python standard library.
- `os.environ`, specifically `REASONS_DB`.
- `os.getcwd`, `os.path.join`, `os.path.exists`, and `os.path.dirname`.
- `api.DEFAULT_DB` from the `reasons.api` module.

## Topics to Explore

- [file] `reasons/mcp_server.py` — Shows how `_find_db` feeds into `_get_db`, `main`, and every MCP tool that calls `_get_db()`.
- [function] `reasons/mcp_server.py:_get_db` — Explains the lazy caching layer around `_find_db` and when discovery happens in server and test lifecycles.
- [file] `tests/test_mcp_server.py` — Likely covers environment overrides, directory walking, fallback behavior, and monkeypatching of `os.getcwd`/`os.path.exists`.
- [general] `REASONS_DB-resolution-semantics` — Worth tracing how the CLI and API resolve database paths, since this function claims to match CLI behavior.
- [file] `reasons/api.py` — Contains `api.DEFAULT_DB` and the downstream code that consumes the returned path, including validation and error behavior.

## Beliefs

- `reasons-db-env-var-has-highest-precedence` — If `REASONS_DB` is set to a non-empty string, `_find_db` returns it without checking filesystem existence.
- `find-db-checks-cwd-and-each-ancestor-including-root` — The discovery loop checks `reasons.db` in the current working directory and every ancestor up to and including the filesystem root.
- `find-db-uses-existence-only-not-database-validation` — `_find_db` uses `os.path.exists`, so it can return a directory, non-SQLite file, or otherwise unusable path named `reasons.db`.
- `find-db-fallback-returns-api-default-db-without-existence-check` — When no override or ancestor match is found, `_find_db` returns `api.DEFAULT_DB` even if that path does not exist.
- `find-db-does-not-expand-user-or-environment-variables` — Values such as `~/reasons.db` or `$HOME/reasons.db` in `REASONS_DB` are returned literally.
