---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: write_contradiction_plan in reasons/api.py

## `write_contradiction_plan`

### 1. Purpose

Serializes a list of detected contradictions into a **human-reviewable markdown "plan" file**. This is the first half of a two-phase workflow for resolving contradictions in the belief network:

1. **Write** a plan here (marked `[APPLY]` by default).
2. **Review** the file by hand — flip `[APPLY]` → `[SKIP]` or delete blocks.
3. **Accept** the reviewed file via `parse_contradiction_plan` → `apply_contradiction_plan`, which calls `add_nogood` for each surviving entry.

The format is explicitly designed to round-trip through `parse_contradiction_plan`, which matches lines of the shape `### NOGOOD <id> [APPLY|SKIP]` and reads the `- Claims:` line beneath it.

### 2. Contract

- **Precondition (per contradiction dict):** must contain `id` (str) and `claims` (non-empty list of str). `severity` and `analysis` are optional. The parser downstream only produces entries when `claims` is non-empty, so an empty `claims` list would round-trip to nothing.
- **Precondition (path):** the parent directory of `output_path` must already exist. There is no `mkdir`.
- **Postcondition:** a file exists at `output_path`. When `append=False`, it is fully replaced and starts with a header + instructions + `---` separator. When `append=True`, blocks are appended without a header.
- **Invariant:** every emitted entry is initially tagged `[APPLY]` so a naive "accept everything" is the default behavior; human review is the safety valve.
- **Idempotence:** not idempotent in append mode (blocks accumulate). Truncate mode is deterministic for a given input.

### 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `contradictions` | `list[dict]` | Detected contradiction records. Each is expected to be a dict from `detect_contradictions` / `contradictions.py`, containing `id`, `claims`, optional `severity`, optional `analysis`. | Missing `id` → `KeyError`. Missing `claims` → `KeyError`. Non-iterable `claims` → `TypeError` from `join`. Non-str claims → `TypeError`. |
| `output_path` | `str` | Destination file path (converted to `Path`). | Must be writable, parent must exist, else `OSError`/`FileNotFoundError`. |
| `append` | `bool` (default `False`) | `False` truncates and writes the header; `True` appends blocks only. | In append mode the header (including the `reasons contradictions --accept ...` instruction) is *not* rewritten — the assumption is a prior call wrote it. Appending to a non-existent file produces a header-less file that still parses. |

### 4. Return value

Returns `str(path)` — the same path the caller passed in, stringified (necessary because `pathlib.Path` is not a string and callers print it back to the user).

The caller is *not* required to handle anything else. Exceptions (from the parameters or from I/O) propagate.

### 5. Algorithm

```
path = Path(output_path)
mode = "a" if append else "w"
open(path, mode) as f:
    if not append:
        write "# Contradiction Plan\n\n"
        write "Review each nogood below. ...\n"
        write "or change APPLY to SKIP. Then run:\n"
        write f"  reasons contradictions --accept {output_path}\n\n"
        write "---\n\n"

    for c in contradictions:
        severity = c.get("severity", "")
        write f"### NOGOOD {c['id']} [APPLY]\n"
        if severity: write f"- Severity: {severity}\n"
        write f"- Claims: {', '.join(c['claims'])}\n"
        if c.get("analysis"): write f"- Analysis: {c['analysis']}\n"
        write "\n"
return str(path)
```

Key format decisions that align with `parse_contradiction_plan`:
- The heading regex is `^###\s+NOGOOD\s+(\S+)\s+\[(APPLY|SKIP)\]` — so `id` **must not contain whitespace**, and the token must be `APPLY` or `SKIP`.
- The claims line must literally begin with `- Claims:`. Claims are split on `,` by the parser — so if any claim ID contains a comma, the round-trip silently corrupts.

### 6. Side effects

- **File I/O only:** opens/writes/closes the target file. No DB access, no network cache mutation, no logging, no stdout.
- In `append` mode each call opens the file once and closes it — so calling this per-batch (as `detect_contradictions` with `output_path` is designed to support) is safe and each flush is a separate handle.

### 7. Error handling

No explicit error handling — all failures are surfaced as exceptions:

- `KeyError` — a dict missing `id` or `claims`.
- `TypeError` — `claims` is not iterable, or elements are not strings (from `', '.join(...)`).
- `FileNotFoundError` / `PermissionError` / `IsADirectoryError` — from `open`.
- `OSError` — general I/O failures.

There is **no validation** that `id` contains no spaces, that `claims` values contain no commas, or that the parent directory exists.

### 8. Usage patterns

- **Typical caller (CLI / interactive):** `detect_contradictions(..., output_path="contradictions.md")` → inspect/edit → `reasons contradictions --accept contradictions.md` → `parse_contradiction_plan` → `apply_contradiction_plan`.
- **Batch/incremental writer:** `detect_contradictions` accepts an `output_path` and passes it into the internal detection routine (in `reasons/contradictions.py`), which presumably calls `write_contradiction_plan(..., append=False)` on the first batch and `append=True` after — exactly why the header is guarded by `if not append`.
- **Caller obligations:**
  - Pass dicts whose `id` values contain no whitespace and whose `claims` values contain no commas (otherwise the parser mis-round-trips).
  - Ensure the parent directory and write permission exist.
  - When using `append=True`, ensure that the file already contains a header from an earlier non-append call (otherwise the resulting file lacks the "how to accept" headline — but is still parseable).

### 9. Dependencies

- `pathlib.Path` (imported at top of `reasons/api.py`).
- Built-in `open`.
- Format coordination with `parse_contradiction_plan` in the *same file* and (transitively) with `apply_contradiction_plan` → `add_nogood` → `Network.add_nogood`.
- Upstream producer shape: dicts from `detect_contradictions` / `contradictions.detect_contradictions[_semantic]`.

### Assumptions not enforced by the type system

- That `id` contains no whitespace — the parser regex is `(\S+)`. An ID with a space breaks round-trip.
- That `claims` items contain no commas — the parser splits on `,`.
- That `id` and `claims` are already deduplicated / safe markdown (no newlines) — nothing escapes newlines, so a claim with `\n` would inject additional lines into the file.
- That the string `[APPLY]`/`[SKIP]` are the only intended states — the parser is case-sensitive.
- That the directory containing `output_path` exists — `Path` does not create it.

## Topics to Explore

- [function] `reasons/api.py:parse_contradiction_plan` — The exact counterpart that reads this format; exercising both is the only way to confirm the round-trip invariant.
- [function] `reasons/api.py:apply_contradiction_plan` — What the reviewed plan actually triggers (`add_nogood`, backtracking, and impact recording).
- [file] `reasons/contradictions.py` — Source of the `contradictions` list, and the caller that likely invokes `write_contradiction_plan(append=True)` per batch when `output_path` is passed to `detect_contradictions`.
- [function] `reasons/api.py:detect_contradictions` — Where `output_path` is threaded through; look for how `append=False` on the first batch transitions to append mode.
- [general] `contradiction-review-workflow` — End-to-end flow: detect → write plan → human edits → parse → apply → nogood backtracking, plus the CLI wiring documented in the file header.

## Beliefs

- `write-contradiction-plan-produces-parseable-format` — The markdown it emits is designed so that `parse_contradiction_plan(Path(output_path).read_text())` reproduces the original `id`/`claims` pairs for every block left as `[APPLY]`.
- `write-contradiction-plan-skips-header-when-appending` — In `append=True` mode the `# Contradiction Plan` title and the `reasons contradictions --accept` instruction are not written; only blocks are appended.
- `write-contradiction-plan-defaults-all-entries-to-apply` — Every contradiction is emitted with an `[APPLY]` marker; exclusion requires human editing to `[SKIP]` or deleting the block.
- `write-contradiction-plan-does-not-create-parent-directory` — It opens the file directly with `open(path, mode)` and never calls `mkdir`, so a missing parent directory raises `FileNotFoundError`.
- `write-contradiction-plan-requires-non-whitespace-ids` — Because the consumer regex in `parse_contradiction_plan` matches `(\S+)` after `NOGOOD`, any `id` containing whitespace silently breaks the round-trip.
