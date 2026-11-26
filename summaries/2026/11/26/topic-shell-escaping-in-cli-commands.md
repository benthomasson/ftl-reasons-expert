---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Only `"` is escaped in the printed command; explore how `reasons update` parses arguments and whether `$`, backticks, or backslashes in `proposed_text` could break the copied command.

## How the command is rendered

The printed command is built in `reasons/propose_update.py` in `format_proposal_markdown`, specifically around lines 256–257:

```python
escaped = proposal["proposed_text"].replace('"', '\\"')
lines.append(f"\n**Command:** `reasons update {nid} \"{escaped}\"`")
```

So the only shell-oriented escaping is:

- every `"` in `proposed_text` becomes `\"`
- the whole text is then wrapped in double quotes
- `nid` is inserted unquoted

There is no use of `shlex.quote` or equivalent in this path. The observed `shlex` usage is only in `reasons/mcp_client.py:10` and `:64`, unrelated to rendering the `reasons update` suggestion. The LLM proposal parser `parse_update_proposals` also does not sanitize `proposed_text` for shell use; it just carries the string through.

## How `reasons update` receives arguments

`pyproject.toml` maps the console command:

```toml
reasons = "reasons.cli:main"
```

`reasons/cli.py` imports `argparse` and defines `cmd_*` handlers that call `reasons.api`, but the provided observation only includes the first 600 lines of `reasons/cli.py` out of 4015. The exact `update` subparser and `cmd_update` implementation are **not present in the observations**, so I cannot verify the precise Python-side argument contract from the supplied snippets.

What is clear is the path when a human copies the printed Markdown command into a POSIX shell:

1. The shell parses and expands the command line.
2. The shell performs variable expansion, command substitution, backslash handling, etc.
3. Only then is `reasons` launched with an already-expanded `argv`.
4. `reasons.cli:main` / argparse then parses those argv strings.

Argparse itself does not interpret `$`, backticks, or shell backslashes. Those are shell-level concerns.

## What can break

Because `proposed_text` is embedded inside shell double quotes, shell double-quote rules apply:

### `$` can expand

If `proposed_text` contains something like:

```text
Cost is $5 or $HOME
```

the copied command becomes:

```sh
reasons update node-id "Cost is $5 or $HOME"
```

The shell may expand `$5` and `$HOME` before `reasons` sees anything. The text passed to `reasons update` is no longer the original proposal. `$(...)` command substitution is also active.

### Backticks can execute commands

If `proposed_text` contains backticks:

```text
Use `date` as the timestamp
```

the shell will execute `date` inside the double-quoted argument. This is both a correctness bug and a command-injection risk. Backticks are not escaped by the current `.replace('"', '\\"')` logic.

Also, because the command is rendered inside Markdown inline code using single backticks:

```python
f"`reasons update {nid} \"{escaped}\"`"
```

a backtick in `proposed_text` can also break the Markdown inline code span, so the displayed command may be malformed before it is even copied.

### Backslashes can alter text or break quoting

The code does not escape backslashes. Inside shell double quotes, backslashes are special before certain characters such as `$`, `` ` ``, `"`, `\`, and newline. Some examples:

- A proposed text ending with `\` can escape the closing quote, leaving the shell string unterminated.
- A proposed text containing `\$HOME` may have the backslash consumed by the shell, changing the intended text.
- A proposed text containing `\\` may collapse to a single backslash.
- A proposed text containing a backslash immediately before a quote is especially problematic: the existing backslash can change the escaping parity of the inserted `\"`, potentially ending the double-quoted argument early.

The quote escaping helps for simple quotes, e.g. bare `"` becomes `\"` and survives inside double quotes, but it is not a complete shell-quoting strategy.

### `nid` is also unquoted

The command interpolates `{nid}` directly. If node IDs are guaranteed to be safe identifiers, that may be fine. If not, `nid` is another injection/breaking point. The observations do not show validation constraints for node IDs.

## What the tests cover

The observed test in `tests/test_propose_update.py` only asserts that the rendered Markdown contains the substring:

```python
assert "reasons update premise-a" in result
```

It does not exercise `$`, backticks, backslashes, embedded quotes, or a trailing backslash. The observed search for CLI update/`proposed_text` usage also found no additional coverage.

## Safer direction

The command should be shell-quoted as a whole, e.g. using `shlex.quote` for both `nid` and `proposed_text`, or the command should be rendered in a fenced code block and explicitly quoted safely. If the command is intended to be copied and pasted into a shell, custom `.replace('"', '\\"')` is insufficient.

## Topics to Explore

- [file] `reasons/cli.py` — Locate the actual `update` subparser and `cmd_update` to verify whether text is positional, option-based, and how it is passed to `api`.
- [function] `reasons/propose_update.py:format_proposal_markdown` — Review the command-rendering logic and replace ad-hoc quote escaping with `shlex.quote` or another complete shell-quoting strategy.
- [general] `shell-quoting` — Test POSIX double-quote behavior with `$HOME`, `$(date)`, backticks, trailing backslashes, and backslash-before-quote sequences.
- [file] `tests/test_propose_update.py` — Add tests covering `proposed_text` containing `$`, backticks, backslashes, quotes, and trailing backslashes.
- [function] `reasons/cli.py:main` — Inspect the argparse registration to confirm the exact `reasons update` argument contract.

## Beliefs

- `propose-update-escapes-only-double-quotes` — `format_proposal_markdown` applies only `proposal["proposed_text"].replace('"', '\\"')` before embedding the text in the suggested command.
- `printed-update-command-uses-double-quote-wrapping` — The generated command wraps `proposed_text` in shell double quotes while leaving `$`, backticks, and backslashes unescaped.
- `reasons-cli-entrypoint-is-main` — The `reasons` console script resolves to `reasons.cli:main` per `pyproject.toml`.
- `shell-expansion-precedes-argparse` — If the printed command is copied into a shell, shell expansion occurs before `reasons.cli` and argparse receive the argument vector.
- `update-parser-not-in-observations` — The observed `reasons/cli.py` excerpt does not include the `update` subparser or `cmd_update`, so the exact Python-side parsing contract is unverified.
