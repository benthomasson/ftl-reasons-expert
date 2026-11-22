---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How callers pass existing `created_at` through updates and whether timezone/format normalization is required elsewhere.

## Short answer

In the observed code, `created_at` is **not passed as a separate update argument**. It rides on the in-memory `Node` object and is written back by `Storage.save()`.

The key flow is:

1. `nodes.created_at` is persisted as a `TEXT` column with default `''` (`reasons/storage.py:29`).
2. `Storage.save()` deletes all node rows and rewrites them (`reasons/storage.py:227` then `reasons/storage.py:234-255`).
3. The insert explicitly includes `created_at` and binds `node.created_at` (`reasons/storage.py:237`, `reasons/storage.py:250`).
4. Therefore, callers preserve `created_at` by loading the node with its existing timestamp, mutating other fields, and saving the same `Node`/`Network`. If they construct a fresh node or leave `created_at` empty, `save()` will write the default empty string — there is no `COALESCE(existing.created_at, incoming)` style preservation in the observed save path.

For network-level metadata, `_init_schema()` sets `network_meta.created_at` only for a new database, using UTC ISO 8601 with second precision (`reasons/storage.py:191-203`). `build_meta()` similarly preserves a supplied `created_at` and generates a UTC ISO timestamp only when it is empty (`reasons/metadata.py:47-52`).

One caution: the visible portion of `Storage.save()` deletes `network_meta` (`reasons/storage.py:231`) and the excerpt ends before showing whether it is reinserted later. The file is 482 lines and only the first 300 were returned, so the network-level restore path is not visible here.

## Timezone and format normalization

The observed code does **not** parse or normalize existing `created_at` values in `Storage.save()`. It writes the string through unchanged.

Observed timestamp generation points:

- New database metadata: `datetime.now(timezone.utc).isoformat(timespec="seconds")` (`reasons/storage.py:192`)
- `build_meta()`: same UTC ISO format for `updated_at`, and `created_at or now` for `created_at` (`reasons/metadata.py:47-52`)
- Report generation: `date.today().isoformat()` for the report display date only (`reasons/report_belief.py:38`) — this is not part of `created_at` persistence.

So:

- If all writes originate from `Storage._init_schema()` and `build_meta()`, timestamps are already canonical UTC ISO 8601 strings with second precision.
- If `created_at` comes from imports, migrations, CLI input, or older data, normalization should happen at that ingestion boundary. The observed save path does not enforce it.
- Because the database type is just `TEXT`, any sorting or comparison of timestamps depends on consistent formatting. Preserving an existing `created_at` unchanged is reasonable when the stored value is already canonical.

## What the observations do not establish

The observations are insufficient to describe the full CLI update path:

- No `Storage.load()` / `load_network()` implementation is shown, so we cannot verify how nodes are rehydrated before updates.
- No CLI update/store/report command implementation is shown.
- `reasons/propose_update.py` generates update proposals but does not apply them and contains no `created_at` usage in the observed excerpt.
- `created_at_usage_in_tests` reports zero matches, so there is no observed test coverage proving timestamp preservation through updates.
- The aggregate search block reports zero `created_at` matches in reasons, but the returned file content clearly contains `created_at` in `reasons/storage.py` and `reasons/metadata.py`. I would treat the search summary as unreliable for this question.

## Topics to Explore

- [file] `reasons/storage.py` — Read the remaining lines after 300 to confirm whether `Storage.save()` restores `network_meta` after deleting it.
- [function] `reasons/storage.py:Storage.save` — Trace exactly which `Node` fields are written and whether any existing-row timestamp fallback exists outside the visible excerpt.
- [file] `reasons/network.py` — Check the `Node` dataclass/defaults and any load/save helpers that populate `created_at`.
- [file] `reasons/cli.py` — Find the actual `update` command path and verify whether it loads the existing node before saving.
- [general] `timestamp-normalization-boundary` — Decide whether imports/migrations should parse and canonicalize timestamps before they reach `Storage.save()`.

## Beliefs

- `storage-save-writes-created-at-from-node-object` — `Storage.save()` binds `node.created_at` directly into the `nodes.created_at` column.
- `storage-save-rewrites-node-rows` — `Storage.save()` deletes all node rows before reinserting them, so preservation depends on the in-memory `Node` carrying the old `created_at`.
- `storage-timestamps-are-text-columns` — `nodes.created_at` and related timestamps are `TEXT` columns with empty-string defaults and no database-level timezone validation.
- `build-meta-preserves-supplied-created-at` — `build_meta()` returns the caller-provided `created_at` if truthy, otherwise a UTC ISO timestamp with second precision.
- `propose-update-does-not-handle-created-at` — The observed `reasons/propose_update.py` content generates proposals but contains no `created_at` handling.
