---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The `{"action": "warn", "target", "value", "timestamp"}` log contract used for observability of skipped dependents; worth verifying against `net.log` consumers.

## The `warn` log contract

`net.log` is treated as an append-only list of event dictionaries. The contract you describe is:

```json
{"action": "warn", "target": ..., "value": ..., "timestamp": ...}
```

For skipped/dangling dependents, `action` is the discriminator literal `"warn"`. The other fields are not optional from a consumer perspective:

- `target` is expected to identify the node/event subject. In the broader log, `target` is used for `add` events too: `tests/test_network.py:354` asserts `e["action"] == "add" and e["target"] == "a"`.
- `value` is the diagnostic payload or message. It must be stringifiable, because `reasons/cli.py:872` interpolates it directly into an f-string.
- `timestamp` is expected to be present on log entries. `reasons/network.py:997` constructs an ISO-8601-style timestamp with `datetime.now().isoformat(timespec="seconds")`, likely for log entries, but the actual `warn` write site is not shown in the observations.

### Consumers that constrain the contract

1. **Warning tests filter by `action == "warn"`**
   - `tests/test_dangling_dependents.py:54`, `64`, `72`, `85`, `94`, `175`, `193`, `205` all do:
     ```python
     warnings = [e for e in net.log if e["action"] == "warn"]
     ```
   - `tests/test_network.py:459` does the same.
   - These tests assume every entry is a dict with an `action` key. They do not, in the observed lines, validate `target`, `value`, or `timestamp` for warn entries.

2. **The CLI assumes all four fields**
   - `reasons/cli.py:872`:
     ```python
     print(f"  {entry['timestamp']}  {entry['action']:10s}  {entry['target']:20s}  {entry['value']}")
     ```
   - This is the strongest consumer-side contract check in the observations: a `warn` entry missing `timestamp`, `target`, or `value` would raise `KeyError` in the CLI log display path.

3. **Storage maps the log fields to a propagation-log schema**
   - SQLite: `reasons/storage.py:344-346` inserts `(entry["timestamp"], entry["action"], entry["target"], entry["value"])` into `propagation_log (timestamp, action, target, value)`.
   - SQLite read-back: `reasons/storage.py:468-472` selects the same four columns and reconstructs `{"timestamp": ts, ...}`.
   - Postgres: `reasons/pg.py:1810-1825` and `reasons/pg.py:3279` use the same `(timestamp, action, target, value)` schema.
   - Therefore, if `net.log` is persisted, warn entries must fit this four-column shape.

4. **API exposes `net.log` directly**
   - `reasons/api.py:1808` does `entries = net.log`.
   - If that API returns entries to clients, the same four-field dictionary shape becomes an external API surface.

5. **The log is heterogeneous**
   - Tests also expect `"add"` actions (`tests/test_network.py:354`, `364`) and `"backtrack"` actions (`tests/test_backtracking.py:184`, `213`).
   - So `warn` is one action variant in a shared log schema. Consumers generally do not guard each entry against missing keys.

### Important verification gap

The observations do **not** include the code that writes the `"warn"` action. The search for skipped-dependents code returned zero matches:

```text
find_skipped_dependents_code: total_count = 0
```

The tests are named `test_dangling_dependents.py`, not skipped-dependents, so the exact semantics of “skipped dependent” are not confirmed here. You should verify:

- Where `net.log.append(...)` or an equivalent `_log(...)` helper writes the warn entry.
- Whether that writer always includes `target`, `value`, and `timestamp`.
- Whether `target` is the skipped dependent ID, the source antecedent ID, or another node ID.
- Whether `value` is a plain string or a structured object. The CLI and SQL storage both favor string/text values.
- Whether the `timestamp` at `reasons/network.py:997` is actually used by the warn path.
- Whether API consumers of `reasons/api.py:1808` depend on the same four keys.

### Bottom line

The contract is stricter than the observed tests imply. The tests mostly care about `action == "warn"`, but the CLI, storage layer, and API exposure assume a full four-field entry:

```json
{"action": "warn", "target": ..., "value": ..., "timestamp": ...}
```

Before changing the warn payload, verify the actual writer and every `net.log` consumer, especially `reasons/cli.py:872` and `reasons/storage.py:344-346`.

## Topics to Explore

- [function] `reasons/network.py:_log` — Find the actual append/write helper for `net.log` and confirm whether it enforces all four fields.
- [file] `reasons/storage.py` — Check propagation-log schema, migrations, and serialization requirements for `value` and `timestamp`.
- [file] `reasons/pg.py` — Compare SQLite and Postgres propagation-log persistence and read-back behavior.
- [file] `tests/test_dangling_dependents.py` — Read beyond the matched warning filters to see which `target`/`value` assertions exist.
- [file] `reasons/api.py` — Inspect how `entries = net.log` at line 1808 is returned or serialized to external callers.

## Beliefs

- `net-log-entries-are-dicts` — `net.log` is treated as a list of dict entries; tests index `e["action"]` directly and call `net.log.clear()`.
- `warn-log-action-is-filtered-by-tests` — Tests in `tests/test_dangling_dependents.py` and `tests/test_network.py` select warnings with `e["action"] == "warn"`.
- `cli-log-display-requires-four-fields` — `reasons/cli.py:872` formats `entry["timestamp"]`, `entry["action"]`, `entry["target"]`, and `entry["value"]` for log entries.
- `propagation-log-schema-maps-net-log-fields` — `reasons/storage.py:344-346` and `reasons/pg.py:3279` persist `net.log`-style entries as `(timestamp, action, target, value)`.
- `warn-writer-not-confirmed` — The observed code search did not locate an `"action": "warn"` write site or skipped-dependents implementation, so the writer must be verified directly.
