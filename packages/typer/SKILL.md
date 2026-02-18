---
name: typer
description: CLI builder using Python type hints, built on Click. Use when creating CLI applications from type-annotated functions, adding subcommands, testing CLIs, or integrating with Rich for formatted output. Triggers on typer, CLI from type hints, typer.run, typer.Typer, CLI builder, command line app.
---

# typer — Type-Hint CLI Builder (v0.15.x)

## Quick Start

```bash
pip install "typer[all]"  # includes rich + shellingham
```

```python
import typer

def main(name: str, count: int = 1, verbose: bool = False):
    """Greet someone."""
    for _ in range(count):
        typer.echo(f"Hello {name}!")

typer.run(main)
# Usage: python app.py Alice --count 3 --verbose
```

## Key Patterns

### Multi-command app
```python
app = typer.Typer()

@app.command()
def create(name: str):
    typer.echo(f"Created {name}")

@app.command()
def delete(name: str, force: bool = False):
    typer.echo(f"Deleted {name}")

app()
```

### Options with metadata
```python
from typing import Annotated

@app.command()
def deploy(
    env: Annotated[str, typer.Argument(help="Target environment")],
    dry_run: Annotated[bool, typer.Option("--dry-run", "-n")] = False,
    port: Annotated[int, typer.Option(min=1, max=65535)] = 8080,
):
    ...
```

## References

- **[api.md](references/api.md)** — Typer app, @command, Argument/Option, parameter types, help text
- **[subcommands.md](references/subcommands.md)** — Command groups, add_typer, callbacks, nested commands
- **[integration.md](references/integration.md)** — Testing with CliRunner, shell completion, Rich integration, packaging
- **[examples.md](references/examples.md)** — Complete examples, gotchas, CLI patterns

## Grep Patterns

- `typer\.Typer\(|app = typer` — Find Typer app definitions
- `@app\.command|@app\.callback` — Find command definitions
- `typer\.Option|typer\.Argument` — Find parameter declarations
