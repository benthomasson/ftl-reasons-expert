---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Trace how `except PermissionError as e:` handlers at `cli.py:521, 644, 746, 816, 834` format the denial for the user

## How the denial is formatted

The relevant CLI code is in `reasons/cli.py`. The top-level imports show only `argparse`, `json`, `os`, `sys`, `pathlib.Path`, `importlib.metadata.version`, and `from . import api`; there is no shared error-formatting helper imported. The observed search for helpers like `print_error`, `format_*_error`, or `Access denied` helpers also found no matches.

Each command handler catches `PermissionError` inline and writes directly to `sys.stderr`, then exits with status `1`.

| Command | API call inside `try` | Handler / print location in observed region | User-facing denial text |
|---|---|---|---|
| `cmd_show` | `api.show_node(...)` | `except PermissionError as e:` around line 520, then line 521 | `Access denied: {e}` |
| `cmd_explain` | `api.explain_node(...)` | `except PermissionError as e:` around line 643, then line 644 | `Access denied: {e}` |
| `cmd_get_metadata` | `api.show_node(...)` | `except (KeyError, PermissionError) as e:` around line 745, then line 746 | `Error: {e}` |
| `cmd_trace_access_tags` | `api.trace_access_tags(...)` | `except PermissionError as e:` around line 815, then line 816 | `Access denied: {e}` |
| `cmd_trace` | `api.trace_assumptions(...)` | `except PermissionError as e:` around line 833, then line 834 | `Access denied: {e}` |

The common pattern is:

```python
except PermissionError as e:
    print(f"Access denied: {e}", file=sys.stderr)
    sys.exit(1)
```

So the user sees the literal prefix `Access denied: ` followed by `str(e)`. The CLI does not add command name, node id, access tag, or any structured JSON/error code. Whatever detail is useful must already be in the `PermissionError` message raised by the `reasons.api` layer.

One important outlier is `cmd_get_metadata`: it catches `PermissionError` together with `KeyError`:

```python
except (KeyError, PermissionError) as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(1)
```

Therefore a permission failure there is presented as `Error: <message>`, not `Access denied: <message>`.

There is a small inconsistency in the observations: the exact search for `except PermissionError as e:` returned `total_count: 0`, while the read region clearly contains such handlers. The line numbers above are therefore based on the observed `reasons/cli.py` region; the exact-match search result should be treated as unreliable or incomplete. Also, `reasons_lib/cli.py` is reported as only 3 lines, so the requested region starting at line 500 is empty and not relevant.

## What the user experiences

For four of the five handlers, the user sees:

```text
Access denied: <exception message from reasons.api>
```

For `cmd_get_metadata`, the user sees:

```text
Error: <exception message from reasons.api>
```

In all cases, the message goes to `stderr`, and the process exits via `sys.exit(1)`.

What is not visible from the observations is the exact `PermissionError` message text produced by `reasons.api`—the CLI formats whatever it receives. The observed CLI-test search also found no matches for permission-denial message expectations, so there is no observed test assertion for these strings.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect the five handlers directly and confirm the exact indentation and line numbers around `cmd_show`, `cmd_explain`, `cmd_get_metadata`, `cmd_trace_access_tags`, and `cmd_trace`.
- [function] `reasons/cli.py:cmd_get_metadata` — Verify whether the combined `(KeyError, PermissionError)` catch should preserve the more specific `Access denied` wording.
- [function] `reasons/api.py:show_node` — Identify what message the API embeds in `PermissionError`, since the CLI prints only `str(e)`.
- [general] `permission-denial-message-tests` — Search for tests asserting stderr output or exit code `1` on `PermissionError`; the observed search found none.
- [file] `reasons_lib/cli.py` — Clarify whether this 3-line file is a stale path, a shim, or unrelated to the active `reasons.cli` implementation.

## Beliefs

- `permission-denials-are-formatted-inline` — Permission handling in `reasons/cli.py` uses inline `print(f"...: {e}", file=sys.stderr)` calls rather than a shared error-formatting helper.
- `access-denied-prefix-used-for-standalone-permission-handlers` — `cmd_show`, `cmd_explain`, `cmd_trace_access_tags`, and `cmd_trace` print `Access denied: {e}` to stderr on `PermissionError`.
- `cmd-get-metadata-maps-permissionerror-to-error-prefix` — `cmd_get_metadata` catches `(KeyError, PermissionError)` together and prints `Error: {e}`, not `Access denied: {e}`.
- `permission-handlers-exit-with-status-one` — All observed permission-denial paths call `sys.exit(1)`.
- `permission-error-message-detail-comes-from-api` — The CLI does not synthesize denial details beyond the prefix; it prints the exception’s string representation verbatim.
