---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How to verify that every `sub.add_parser(...)` name has a matching entry in the `commands` dict; current code relies on discipline rather than automated checks.

## How to verify the `sub.add_parser(...)` / `commands` invariant

### Current state

There are two independent pieces of CLI wiring in `reasons/cli.py`:

1. Parser registration:
   - `parser = argparse.ArgumentParser(...)` at `reasons/cli.py:3053`
   - `sub = parser.add_subparsers(dest="command")` at `reasons/cli.py:3065`
   - individual commands registered via calls like:
     - `sub.add_parser("init", ...)` at `reasons/cli.py:3068`
     - `sub.add_parser("add", ...)` at `reasons/cli.py:3073`
     - and many more through `reasons/cli.py:3270`

2. Dispatch table:
   - `commands = {` at `reasons/cli.py:3923`

The current test suite exercises CLI behavior through `main()` by patching `sys.argv`, as seen in `tests/test_cli.py` (`run_cli` helper around lines 16-31). Those tests cover selected commands, but there is no test that structurally compares every `sub.add_parser(...)` name with the keys of `commands`. So if someone adds:

```python
sub.add_parser("new-command", ...)
```

and forgets:

```python
commands = {
    ...,
    "new-command": cmd_new_command,
}
```

the parser will accept `reasons new-command`, but dispatch will not have a handler. Existing tests may still pass if no test invokes `new-command`.

Also, `reasons_lib/cli.py` is only a re-export:

```python
from reasons.cli import *  # noqa: F401,F403
```

so the source of truth to check is `reasons/cli.py`.

### Recommended automated check: static AST test

Add a structural test that parses `reasons/cli.py` and extracts:

- all first arguments to `sub.add_parser(...)`
- all string keys in the `commands = { ... }` dict literal

Then assert that every parser name appears in `commands`.

Example test for `tests/test_cli.py`:

```python
import ast
from pathlib import Path


CLI_PATH = Path(__file__).resolve().parents[1] / "reasons" / "cli.py"


def _collect_cli_registration():
    tree = ast.parse(CLI_PATH.read_text())

    subparser_names = set()
    command_keys = set()
    nonliteral_subparsers = []
    nonliteral_command_keys = []

    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            fn = node.func
            if (
                isinstance(fn, ast.Attribute)
                and fn.attr == "add_parser"
                and isinstance(fn.value, ast.Name)
                and fn.value.id == "sub"
            ):
                if (
                    node.args
                    and isinstance(node.args[0], ast.Constant)
                    and isinstance(node.args[0].value, str)
                ):
                    subparser_names.add(node.args[0].value)
                else:
                    nonliteral_subparsers.append(ast.unparse(node))

        if isinstance(node, ast.Assign):
            for target in node.targets:
                if (
                    isinstance(target, ast.Name)
                    and target.id == "commands"
                    and isinstance(node.value, ast.Dict)
                ):
                    for key in node.value.keys:
                        if (
                            isinstance(key, ast.Constant)
                            and isinstance(key.value, str)
                        ):
                            command_keys.add(key.value)
                        elif key is not None:
                            nonliteral_command_keys.append(ast.unparse(key))

    return (
        subparser_names,
        command_keys,
        nonliteral_subparsers,
        nonliteral_command_keys,
    )


def test_all_subparsers_have_command_handlers():
    (
        subparser_names,
        command_keys,
        nonliteral_subparsers,
        nonliteral_command_keys,
    ) = _collect_cli_registration()

    assert not nonliteral_subparsers, (
        f"Non-literal sub.add_parser names found: {nonliteral_subparsers}"
    )
    assert not nonliteral_command_keys, (
        f"Non-literal commands dict keys found: {nonliteral_command_keys}"
    )

    missing = subparser_names - command_keys
    assert not missing, (
        f"Subparsers with no commands entry: {sorted(missing)}"
    )
```

This directly enforces the invariant:

```python
set(sub.add_parser names) <= set(commands keys)
```

If you also want to catch stale handler entries, add:

```python
assert not (command_keys - subparser_names), (
    f"commands entries with no subparser: {sorted(command_keys - subparser_names)}"
)
```

but be careful if you intentionally support aliases or hidden commands.

### Alternative: runtime assertion

If `sub` and `commands` are in the same scope inside `main()`, you can add a guard immediately after the `commands` dict is defined:

```python
missing = set(sub.choices) - set(commands)
if missing:
    parser.error(
        f"internal error: no CLI handler for command(s): {sorted(missing)}"
    )
```

Here `sub.choices` is argparse’s mapping of registered subcommand names to subparsers. This catches the problem on every CLI startup, not only when the missing command is invoked.

The static AST test is preferable because:

- it runs in normal `pytest` without building a database,
- it does not depend on `main()` execution,
- it can fail fast in CI before packaging/release.

### Caveats from the provided observations

The supplied snapshot is not enough to produce the actual list of missing handlers:

- `find_add_parser_calls` reports `total_count: 89`, but the `matches` array shown contains only 30 entries.
- The body of the `commands` dict at `reasons/cli.py:3923` is not included.
- The parser construction region around lines `3053-3270` is not in the returned `read_primary_cli` excerpt; those lines are only known from the search matches.

So the method above is the verification. The current data cannot conclusively tell you whether all 89 parser names are already in `commands`.

## Topics to Explore

- [file] `reasons/cli.py` — Contains parser registration around lines 3053-3270 and the dispatch table at line 3923; this is where the invariant must hold.
- [function] `reasons/cli.py:main` — Likely contains parser construction, `commands` definition, and dispatch; inspect how `args.command` is used.
- [file] `tests/test_cli.py` — Existing CLI tests exercise behavior through `main()` but do not structurally compare parser names to `commands` keys.
- [general] `argparse-subparser-choices` — Understand how `sub.choices` maps registered subcommand names to argparse parsers for runtime checks.
- [general] `static-ast-testing` — Useful pattern for enforcing codebase invariants between two distant declarations without running the CLI.

## Beliefs

- `commands-dict-is-defined-at-reasons-cli-line-3923` — `reasons/cli.py` contains the only `commands = {` assignment at line 3923.
- `subparser-group-is-created-at-reasons-cli-line-3065` — `reasons/cli.py` calls `parser.add_subparsers(dest="command")` at line 3065.
- `cli-tests-exercise-commands-through-main` — `tests/test_cli.py` invokes `main()` by patching `sys.argv`, not by checking parser-to-handler registration.
- `reasons-lib-cli-is-a-re-export` — `reasons_lib/cli.py` imports everything from `reasons.cli` and adds no separate command registration.
- `observed-add-parser-calls-use-string-literal-names` — The 30 listed `sub.add_parser(...)` calls pass literal command names such as `"init"` and `"add"`.
