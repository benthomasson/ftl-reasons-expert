---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Search for the actual CLI/API update code to verify whether any update path accepts and persists `example`.

## Search result: `example` is accepted by the add path, not by any visible update path

The observed code verifies that `example` is accepted and persisted on **node creation**, but the provided slices do **not** show any update path that accepts `example`.

### What is verified

- CLI `add` exposes `--example`:
  - `reasons/cli.py:3085`:
    ```python
    p.add_argument("--example", default=None, help="Code example demonstrating the belief")
    ```
- `cmd_add` forwards it into the API:
  - `reasons/cli.py:59`:
    ```python
    example=getattr(args, "example", None),
    ```
- `api.add_node` accepts `example`:
  - `reasons/api.py:289`:
    ```python
    example: str | None = None,
    ```
- `api.add_node` persists it into node metadata:
  - `reasons/api.py:379-380`:
    ```python
    if example is not None:
        metadata["example"] = example
    ```
- The PostgreSQL add path does the same:
  - `reasons/pg.py:217-223` accepts `example` and stores `metadata["example"]`.
- `show` reads and prints it:
  - `reasons/cli.py:556-557` checks `node["metadata"].get("example")` and prints an `Example:` section.

Persistence is metadata-based: `Storage.save` serializes `node.metadata` into `metadata_json`, and `Storage.load` deserializes it back. So once `metadata["example"]` is set, it persists through the normal storage path.

### Update-path search

The update-related definitions found were:

- `reasons/api.py:2813` — `pin_update`
- `reasons/api.py:4458` — `propose_update`
- `reasons/cli.py:1218` — `cmd_pin_update`
- `reasons/cli.py:2775` — `cmd_propose_update`
- `reasons/mcp_server.py:268` — `propose_update`
- `reasons/propose_update.py:291` — `propose_updates`

None of the visible update signatures include an `example` parameter. The proposal-update code works with fields such as `action`, `proposed_text`, `failure_mode`, `basis`, `evidence`, and `comment`; it does not appear to carry or persist `example`.

Also notable: `reasons/propose_update.py` formats a suggested command like:

```text
reasons update <nid> "<proposed_text>"
```

but the update-definition search does **not** show a `cmd_update` definition. That suggests either the suggested command is stale/unimplemented, or the actual handler was not captured by the search. The CLI slice provided only covers `reasons/cli.py:1-1000`, so `cmd_pin_update` at line 1218 and `cmd_propose_update` at line 2775 were not read directly.

### What is missing for a definitive answer

To conclusively verify every update path, we would need:

- `reasons/cli.py` around lines 1218 and 2775 to inspect `cmd_pin_update` and `cmd_propose_update`.
- Any parser registration for an actual `update` command, if it exists.
- `reasons/api.py` around lines 2813 and 4458 to inspect `pin_update` and `propose_update`.
- Full `find_example_references` results. The observation reports `total_count: 51` but only shows 30 matches, so the match list is truncated.
- PostgreSQL update methods beyond the visible `pg.py:217-223` add path.

### Bottom line

Based on the provided observations: **the only verified CLI/API path that accepts and persists `example` is the add path via `api.add_node` / CLI `add --example`.** No visible update function or update handler accepts `example`. However, because the actual bodies of the update handlers and the full `example` search results were not included, this should be confirmed by reading the update command handlers and full grep results.

## Topics to Explore

- [function] `reasons/cli.py:cmd_pin_update` — Inspect whether this update handler accepts `--example` or mutates node metadata directly.
- [function] `reasons/cli.py:cmd_propose_update` — Check whether the proposal-update CLI path carries or persists `example`.
- [function] `reasons/api.py:propose_update` — Verify whether the API proposal-update path has any metadata or `example` parameter.
- [file] `reasons/pg.py` — Check PostgreSQL update methods for an `example` parameter or metadata merge that could accept it.
- [general] `full-example-grep-results` — Re-run the `example` search and inspect all 51 matches, since the observation only shows 30.

## Beliefs

- `add-node-persists-example-in-metadata` — `api.add_node` stores non-`None` `example` as `node.metadata["example"]` at `reasons/api.py:379-380`.
- `cli-add-exposes-example-flag` — `reasons add` registers `--example` at `reasons/cli.py:3085` and forwards it through `cmd_add` at `reasons/cli.py:59`.
- `metadata-json-is-persistence-path-for-example` — `Storage.save` serializes `node.metadata` to `metadata_json`, and `Storage.load` deserializes it, so metadata keys such as `example` persist.
- `no-visible-update-signature-accepts-example` — The discovered update functions (`pin_update`, `propose_update`, `propose_updates`) and CLI handlers do not expose an `example` parameter in the observed code.
- `propose-update-emits-update-command` — `format_proposal_markdown` in `reasons/propose_update.py` suggests `reasons update ...` commands, but no matching `cmd_update` definition appears in the update-definition search.
