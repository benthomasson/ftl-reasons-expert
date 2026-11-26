---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: read_source in reasons/verify.py

`read_source` reads the on-disk source document associated with a belief so that `verify.py` can send the actual source text to an LLM for verification. It intentionally delegates path interpretation to `check_stale.resolve_source_path` so verification and stale-checking agree on how a `source` string maps to a file.

## Purpose

- Load the source document referenced by a belief.
- Resolve the source path using the same infrastructure used for stale checks.
- Degrade gracefully to `None` when the source cannot be found or read.
- Truncate very large files before they are embedded in an LLM prompt.

This exists so verification can distinguish "source is available and contains this text" from "source is missing/unreadable", without duplicating path-resolution logic.

## Contract

### Preconditions

- `source` must be a value that `resolve_source_path` understands, typically a source path string from a belief.
- `db_path`, if provided, is assumed to be a path to a database file, not a directory, because only its parent directory is used.
- `resolve_source_path` is assumed to return a `pathlib.Path`-like object or a falsy value such as `None`.
- `MAX_SOURCE_CHARS` must exist and be an integer.

### Postconditions

- Returns either:
  - a string containing the source file text, or
  - `None` if the resolved path is missing, falsy, non-existent, unreadable, or otherwise fails with an `OSError`.
- If the original text length is greater than `MAX_SOURCE_CHARS`, the returned string is the first `MAX_SOURCE_CHARS` characters plus `"\n\n[... truncated ...]"`.
- If the file is empty, returns `""`, not `None`.

### Invariant / caveat

The docstring says the content is "truncated to `MAX_SOURCE_CHARS`", but when truncation occurs the returned value is actually longer than `MAX_SOURCE_CHARS` because the truncation marker is appended. Callers should not assume `len(result) <= MAX_SOURCE_CHARS`.

## Parameters

### `source`

The source identifier/path to resolve. Expected to be compatible with `resolve_source_path`. The function does not validate the type or format itself.

Edge cases:

- `None` or malformed values may cause `resolve_source_path` to return `None` or raise, depending on its implementation.
- Relative paths are interpreted by `resolve_source_path`, with `db_dir` as contextual base when provided.
- URL-like sources are not handled directly by this function; only whatever `resolve_source_path` maps to a local path can be read.

### `db_path=None`

Optional path to the database or metadata file. The function does not use `db_path` directly as a directory; it computes:

```python
db_dir = Path(db_path).parent if db_path else None
```

So `db_path` is assumed to be a file path. If a directory path is passed, its parent directory is used, which is probably not intended.

Edge cases:

- `None` or empty string: `db_dir` is `None`.
- Relative filename such as `"beliefs.db"`: `db_dir` becomes `"."`.
- Invalid truthy object: `Path(db_path)` may raise `TypeError`.

## Return Value

- `None`: source path could not be resolved, does not exist, or could not be read.
- `str`: source content. This may be empty (`""`) for an empty file.
- Truncated `str`: first `MAX_SOURCE_CHARS` characters of the original file plus a truncation marker.

The caller must handle `None`. The caller should also be aware that `""` is a valid return value but may be treated as "missing" by downstream formatting code. For example, `format_belief_for_verify` uses:

```python
source_content or "(source not available)"
```

so an empty string is rendered as unavailable there.

## Algorithm

1. Lazily import `resolve_source_path` from `.check_stale`.
   - This avoids importing `check_stale` until `read_source` is actually called.
   - It also means import-time errors in `check_stale` surface at call time.

2. Compute `db_dir`:
   - If `db_path` is truthy, take its parent directory.
   - Otherwise use `None`.

3. Resolve the source:
   ```python
   path = resolve_source_path(source, db_dir=db_dir)
   ```
   The resolver is responsible for interpreting relative paths, database-relative paths, or other source-string conventions.

4. If `path` is falsy or does not exist, return `None`.

5. Read the file:
   ```python
   content = path.read_text(errors="replace")
   ```
   This reads the entire file into memory. Invalid byte sequences are replaced rather than raising decoding errors in normal operation.

6. If reading fails with `OSError` or `UnicodeDecodeError`, return `None`.

7. If `len(content) > MAX_SOURCE_CHARS`, truncate to the first `MAX_SOURCE_CHARS` characters and append:
   ```text
   \n\n[... truncated ...]
   ```

8. Return the content string.

## Side Effects

- Lazily imports `reasons.check_stale.resolve_source_path`.
- Performs filesystem existence checking via `path.exists()`.
- Reads the entire file from disk via `path.read_text(...)`.
- Does not write, mutate, or cache anything.
- May trigger whatever side effects `resolve_source_path` has, depending on its implementation.

## Error Handling

### Swallowed / converted to `None`

- `OSError` during `read_text`: returns `None`.
  - Includes `FileNotFoundError`, `PermissionError`, `IsADirectoryError`, and other filesystem read failures.
- `UnicodeDecodeError` during `read_text`: returns `None`.
  - In practice, `errors="replace"` makes decoding errors unlikely, but the catch is defensive.

### Not caught

- Exceptions from `Path(db_path)` if `db_path` is invalid.
- Exceptions from `resolve_source_path`.
- Exceptions from `path.exists()` if the returned object is malformed or if `exists()` raises.
- `AttributeError` if `resolve_source_path` returns a non-Path truthy object without `.exists` / `.read_text`.
- `MemoryError` if reading an extremely large file exhausts memory.
- `KeyboardInterrupt`, `SystemExit`, etc.

So `read_source` is not a "never raises" function. It only converts filesystem read failures into `None`.

## Usage Patterns

Typical usage:

```python
content = read_source(belief["source"], db_path=db_path)
```

Callers should:

- Check for `None` before assuming source text exists.
- Pass the same `db_path` context used elsewhere so relative sources resolve consistently.
- Treat `""` as a valid but empty source file.
- Not assume the returned string is always at most `MAX_SOURCE_CHARS` characters.
- Avoid using it for enormous files if memory usage matters, because it reads the whole file before truncating.

## Dependencies

- `.check_stale.resolve_source_path` for path resolution.
- `pathlib.Path` for `db_path` parent calculation.
- Module constant `MAX_SOURCE_CHARS`.
- Standard filesystem APIs via `Path.exists()` and `Path.read_text()`.

## Assumptions Not Enforced by Types

- `source` is a format understood by `resolve_source_path`.
- `db_path` is a file path, not a directory path.
- `resolve_source_path` returns a `pathlib.Path` or `None`, not an arbitrary string.
- The source is a local filesystem path after resolution.
- Reading the entire file into memory is acceptable.
- The system default text encoding is acceptable; no encoding is explicitly specified.
- The file is expected to be text, though directories and unreadable files are handled via `OSError`.
- `MAX_SOURCE_CHARS` is a sensible non-negative integer.

## Topics to Explore

- [function] `reasons/check_stale.py:resolve_source_path` — Understand how `source` strings and `db_dir` are converted into filesystem paths.
- [file] `tests/test_verify.py` — Check expected behavior for missing, unreadable, empty, and truncated source files.
- [function] `reasons/verify.py:format_belief_for_verify` — See how `None` and `""` source content are presented to the LLM.
- [general] `source-path-resolution` — Explore relative vs. absolute source semantics and database-relative path handling.
- [file] `reasons/verify.py` — See how `read_source` fits into the broader verification workflow.

## Beliefs

- `read-source-returns-none-for-missing-or-unreadable-source` — `read_source` returns `None` when the resolved path is falsy, does not exist, or fails with an `OSError` during reading.
- `read-source-uses-db-path-parent-as-db-dir` — When `db_path` is provided, `read_source` passes `Path(db_path).parent` to `resolve_source_path` rather than using `db_path` directly.
- `read-source-reads-full-file-before-truncating` — The function calls `path.read_text()` on the entire file and only then truncates based on `MAX_SOURCE_CHARS`.
- `read-source-truncation-output-exceeds-max-source-chars` — Truncated results contain the first `MAX_SOURCE_CHARS` characters plus an appended `"\n\n[... truncated ...]"` marker, so the returned length exceeds `MAX_SOURCE_CHARS`.
- `read-source-returns-empty-string-for-empty-file` — An existing empty source file produces `""`, not `None`.
