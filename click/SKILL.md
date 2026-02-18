---
name: click
description: Composable command-line interface framework. Use when building CLI applications with commands, options, arguments, groups, or testing CLIs with CliRunner. Foundation for Typer. Triggers on click, CLI framework, command line, click.command, click.option, click.group, CliRunner.
---

# click — CLI Framework (v8.1.8)

## Quick Start

```bash
pip install click
```

```python
import click

@click.command()
@click.option("--name", "-n", required=True, help="Your name")
@click.option("--count", "-c", default=1, help="Repetitions")
def greet(name, count):
    """Greet someone."""
    for _ in range(count):
        click.echo(f"Hello, {name}!")

if __name__ == "__main__":
    greet()
```

## Key Patterns

### Command with options and arguments
```python
@click.command()
@click.option("--verbose", is_flag=True)
@click.option("--mode", type=click.Choice(["fast", "safe"]))
@click.argument("name")
def process(name, verbose, mode):
    click.echo(f"Processing {name}")
```

### Groups and subcommands
```python
@click.group()
def cli():
    pass

@cli.command()
@click.argument("name")
def create(name):
    click.echo(f"Created {name}")
```

### Testing with CliRunner
```python
from click.testing import CliRunner
runner = CliRunner()
result = runner.invoke(cli, ["create", "foo"])
assert result.exit_code == 0
```

## References

- **[api.md](references/api.md)** — @command, @option, @argument, parameter types, decorators
- **[commands.md](references/commands.md)** — Groups, chained commands, Context, pass_context, plugins
- **[testing.md](references/testing.md)** — CliRunner, invoke testing, utilities (echo, prompt, confirm)
- **[examples.md](references/examples.md)** — Complete examples, gotchas, decorator ordering

## Grep Patterns

- `@click\.command|@click\.group` — Find command definitions
- `@click\.option|@click\.argument` — Find parameter declarations
- `CliRunner` — Find test invocations
