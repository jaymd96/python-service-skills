# Typer — Examples & Gotchas

> Part of the typer skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Boolean Flags](#1-boolean-flags----flag----no-flag)
  - [2. Optional vs Required with None Defaults](#2-optional-vs-required-with-none-defaults)
  - [3. List Arguments and Options](#3-list-arguments-and-options)
  - [4. Async Commands](#4-async-commands)
  - [5. Click Compatibility Edge Cases](#5-click-compatibility-edge-cases)
  - [6. Argument vs Option Confusion](#6-argument-vs-option-confusion)
  - [7. Single Command Apps With Explicit Typer](#7-single-command-apps-with-explicit-typer)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Simple Single Command](#example-1-simple-single-command)
  - [Example 2: Multi-Command App with Subgroups](#example-2-multi-command-app-with-subgroups)
  - [Example 3: Options with Validation](#example-3-options-with-validation)
  - [Example 4: File Path Arguments](#example-4-file-path-arguments)
  - [Example 5: Interactive Prompts](#example-5-interactive-prompts)
  - [Example 6: Testing with CliRunner](#example-6-testing-with-clirunner)
  - [Example 7: Rich-Formatted Output](#example-7-rich-formatted-output)
- [Sources](#sources)

## Gotchas and Common Mistakes

### 1. Boolean Flags (`--flag` / `--no-flag`)

A `bool` parameter with a default automatically creates a flag pair:

```python
def main(verbose: bool = False):
    ...
# Creates: --verbose / --no-verbose
```

To customize flag names:

```python
def main(
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Enable verbose mode"),
):
    ...
# Creates: --verbose / -v (and --no-verbose)
```

To create a `--flag / --no-flag` pair with custom names:

```python
def main(
    cache: bool = typer.Option(True, "--cache/--no-cache", help="Enable caching"),
):
    ...
```

**Gotcha**: If you set the default to `True`, the flag logic is inverted -- the user must pass `--no-flag` to disable it.

### 2. Optional vs Required with `None` Defaults

```python
# REQUIRED argument (no default):
def main(name: str):
    ...

# OPTIONAL with None (must use Optional or | None):
def main(name: Optional[str] = None):
    ...

# GOTCHA: This makes it an Option (not Argument) with None default:
def main(name: Optional[str] = typer.Option(None)):
    ...

# GOTCHA: This is REQUIRED despite Optional type hint, because of ...:
def main(name: Optional[str] = typer.Option(...)):
    ...
```

**Rule of thumb**: The *default value* (not the type hint) determines whether a parameter is required. `...` (Ellipsis) means required; any other value (including `None`) means optional.

### 3. List Arguments and Options

Lists require explicit `typer.Option()` or `typer.Argument()`:

```python
from typing import List, Optional

# CORRECT: List option
def main(items: Optional[List[str]] = typer.Option(None)):
    ...
# Usage: --items a --items b --items c

# CORRECT: List argument (consumes remaining positional args)
def main(files: List[str] = typer.Argument(None)):
    ...
# Usage: app.py file1.txt file2.txt file3.txt

# GOTCHA: This will NOT work as expected without typer.Option():
def main(items: List[str] = []):  # typer cannot infer this properly
    ...
```

### 4. Async Commands

Typer does **not** natively support async command functions. If you need async:

```python
import asyncio
import typer

app = typer.Typer()

async def async_deploy(env: str):
    """The actual async logic."""
    await asyncio.sleep(1)
    typer.echo(f"Deployed to {env}")

@app.command()
def deploy(env: str):
    """Synchronous wrapper for async deploy."""
    asyncio.run(async_deploy(env))
```

Alternatively, use the `anyio` or `asyncer` library for a cleaner pattern, but the sync wrapper approach is the most common.

### 5. Click Compatibility Edge Cases

- **Click's `@click.group()` vs Typer's `Typer()`**: They are not directly interchangeable. Use `typer.main.get_command(app)` to convert Typer to Click.
- **Click parameter decorators**: Mixing `@click.option()` with Typer commands can cause issues. Use `typer.Option()` instead.
- **Click `File` type**: Use `typer.FileText` or `typer.FileBinaryRead` etc. instead of `click.File()`.
- **Click `Context.invoke()`**: Works but can behave unexpectedly with Typer's type conversion layer.
- **Middleware / plugins**: When building a Click plugin ecosystem, convert your Typer app via `get_command()`.

### 6. Argument vs Option Confusion

In typer, the distinction is driven by how you annotate:

```python
# ARGUMENT (positional): no default or uses typer.Argument()
def main(name: str):                              # positional argument
    ...
def main(name: str = typer.Argument("world")):    # positional with default
    ...

# OPTION (--flag): has a plain default value or uses typer.Option()
def main(name: str = "world"):                    # option: --name
    ...
def main(name: str = typer.Option("world")):      # option: --name
    ...
```

**Gotcha**: A bare default value (e.g., `name: str = "world"`) becomes an **option**, not an argument with a default. Use `typer.Argument("world")` if you want a positional argument with a default.

### 7. Single Command Apps With Explicit Typer

When a `Typer()` app has only one command, typer automatically hides the subcommand layer:

```python
app = typer.Typer()

@app.command()
def main(name: str):
    typer.echo(f"Hello, {name}")

# Usage: python app.py Alice (not: python app.py main Alice)
```

This is generally desired behavior, but can be surprising if you plan to add more commands later.

## Complete Code Examples

### Example 1: Simple Single Command

```python
"""greet.py -- A simple greeting CLI."""
import typer

def main(
    name: str = typer.Argument(..., help="Person to greet"),
    greeting: str = typer.Option("Hello", "--greeting", "-g", help="Greeting word"),
    count: int = typer.Option(1, "--count", "-c", min=1, max=10, help="Repetitions"),
    shout: bool = typer.Option(False, "--shout", "-s", help="UPPERCASE output"),
):
    """Greet someone with a customizable message."""
    for _ in range(count):
        message = f"{greeting}, {name}!"
        if shout:
            message = message.upper()
        typer.echo(message)

if __name__ == "__main__":
    typer.run(main)
```

```bash
$ python greet.py World -g "Hi" -c 3 --shout
HI, WORLD!
HI, WORLD!
HI, WORLD!
```

### Example 2: Multi-Command App with Subgroups

```python
"""deploy.py -- A multi-command deployment tool."""
import typer
from enum import Enum
from typing import Optional, List

# -- Root app --
app = typer.Typer(
    help="Deployment management CLI",
    no_args_is_help=True,
    rich_markup_mode="rich",
)

# -- Service subgroup --
svc_app = typer.Typer(help="Manage services")
app.add_typer(svc_app, name="service")

# -- Config subgroup --
cfg_app = typer.Typer(help="Manage configuration")
app.add_typer(cfg_app, name="config")


class Environment(str, Enum):
    dev = "dev"
    staging = "staging"
    production = "production"


# -- Root callback (global options) --
@app.callback()
def main(
    ctx: typer.Context,
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Enable verbose output"),
    dry_run: bool = typer.Option(False, "--dry-run", help="Simulate without making changes"),
):
    """
    [bold]Deploy CLI[/bold] -- Manage services and configuration.
    """
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose
    ctx.obj["dry_run"] = dry_run


# -- Service commands --
@svc_app.command()
def deploy(
    ctx: typer.Context,
    service: str = typer.Argument(..., help="Service name"),
    env: Environment = typer.Option(Environment.dev, help="Target environment"),
    tag: str = typer.Option("latest", help="Docker image tag"),
):
    """Deploy a service to an environment."""
    prefix = "[DRY RUN] " if ctx.obj.get("dry_run") else ""
    typer.echo(f"{prefix}Deploying {service}:{tag} to {env.value}")

@svc_app.command()
def rollback(
    service: str = typer.Argument(..., help="Service name"),
    env: Environment = typer.Option(..., help="Target environment"),
    version: Optional[str] = typer.Option(None, help="Specific version to rollback to"),
):
    """Rollback a service to a previous version."""
    target = version or "previous"
    typer.confirm(f"Rollback {service} in {env.value} to {target}?", abort=True)
    typer.secho(f"Rolling back {service} to {target}", fg=typer.colors.YELLOW)

@svc_app.command("list")
def list_services(
    env: Environment = typer.Option(Environment.dev, help="Environment to list"),
):
    """List all deployed services."""
    typer.echo(f"Services in {env.value}:")
    typer.echo("  - api (v2.3.1)")
    typer.echo("  - web (v1.8.0)")
    typer.echo("  - worker (v2.3.1)")


# -- Config commands --
@cfg_app.command()
def show(
    service: str = typer.Argument(..., help="Service name"),
    env: Environment = typer.Option(Environment.dev, help="Target environment"),
):
    """Show configuration for a service."""
    typer.echo(f"Config for {service} in {env.value}:")
    typer.echo('  database_url: "postgres://..."')
    typer.echo("  log_level: INFO")

@cfg_app.command("set")
def set_config(
    service: str = typer.Argument(..., help="Service name"),
    key: str = typer.Argument(..., help="Config key"),
    value: str = typer.Argument(..., help="Config value"),
    env: Environment = typer.Option(Environment.dev, help="Target environment"),
):
    """Set a configuration value for a service."""
    typer.confirm(f"Set {key}={value} for {service} in {env.value}?", abort=True)
    typer.secho(f"Updated {key} for {service}", fg=typer.colors.GREEN)


if __name__ == "__main__":
    app()
```

```bash
$ python deploy.py --help
$ python deploy.py service deploy api --env production --tag v2.4.0
$ python deploy.py --dry-run service deploy api --env staging
$ python deploy.py config set api log_level DEBUG --env dev
```

### Example 3: Options with Validation

```python
"""validated.py -- Demonstrating parameter validation."""
import typer
from typing import Annotated
from pathlib import Path
from enum import Enum


class LogLevel(str, Enum):
    debug = "debug"
    info = "info"
    warning = "warning"
    error = "error"


def validate_name(value: str) -> str:
    if not value.isalpha():
        raise typer.BadParameter("Name must contain only letters")
    if len(value) < 2:
        raise typer.BadParameter("Name must be at least 2 characters")
    return value.capitalize()


app = typer.Typer()

@app.command()
def create(
    name: Annotated[str, typer.Argument(help="Project name", callback=validate_name)],
    port: Annotated[int, typer.Option(min=1024, max=65535, help="Server port")] = 8080,
    workers: Annotated[int, typer.Option(min=1, max=32, clamp=True, help="Number of workers")] = 4,
    log_level: Annotated[LogLevel, typer.Option(help="Logging level")] = LogLevel.info,
    output_dir: Annotated[Path, typer.Option(
        help="Output directory",
        exists=False,
        file_okay=False,
        dir_okay=True,
        resolve_path=True,
    )] = Path("./output"),
):
    """Create a new project with validated settings."""
    typer.echo(f"Project: {name}")
    typer.echo(f"Port: {port}")
    typer.echo(f"Workers: {workers}")
    typer.echo(f"Log level: {log_level.value}")
    typer.echo(f"Output: {output_dir}")


if __name__ == "__main__":
    app()
```

```bash
$ python validated.py myproject --port 3000 --workers 8 --log-level debug
Project: Myproject
Port: 3000
Workers: 8
Log level: debug
Output: /absolute/path/to/output

$ python validated.py "ab123" --port 80
Error: Invalid value for 'NAME': Name must contain only letters
```

### Example 4: File Path Arguments

```python
"""fileops.py -- File operations with Path validation."""
import typer
from pathlib import Path
from typing import Optional


app = typer.Typer()

@app.command()
def convert(
    input_file: Path = typer.Argument(
        ...,
        help="Input file to process",
        exists=True,
        file_okay=True,
        dir_okay=False,
        readable=True,
        resolve_path=True,
    ),
    output_file: Path = typer.Argument(
        ...,
        help="Output file path",
        file_okay=True,
        dir_okay=False,
        writable=True,
        resolve_path=True,
    ),
    encoding: str = typer.Option("utf-8", help="File encoding"),
):
    """Convert a file from one format to another."""
    content = input_file.read_text(encoding=encoding)
    typer.echo(f"Read {len(content)} characters from {input_file.name}")

    output_file.write_text(content.upper(), encoding=encoding)
    typer.secho(f"Wrote to {output_file}", fg=typer.colors.GREEN)


@app.command()
def find(
    directory: Path = typer.Argument(
        ".",
        help="Directory to search",
        exists=True,
        file_okay=False,
        dir_okay=True,
        resolve_path=True,
    ),
    pattern: str = typer.Option("*", "--pattern", "-p", help="Glob pattern"),
    max_depth: Optional[int] = typer.Option(None, help="Maximum directory depth"),
):
    """Find files matching a pattern."""
    glob_method = directory.rglob if max_depth is None else directory.glob
    matches = list(glob_method(pattern))
    typer.echo(f"Found {len(matches)} files matching '{pattern}':")
    for match in matches[:20]:
        typer.echo(f"  {match}")
    if len(matches) > 20:
        typer.echo(f"  ... and {len(matches) - 20} more")


if __name__ == "__main__":
    app()
```

```bash
$ python fileops.py convert input.txt output.txt --encoding utf-8
Read 1234 characters from input.txt
Wrote to /path/to/output.txt

$ python fileops.py find ./src --pattern "*.py"
Found 15 files matching '*.py':
  ./src/main.py
  ./src/utils.py
  ...
```

### Example 5: Interactive Prompts

```python
"""interactive.py -- Interactive user input."""
import typer
from enum import Enum
from typing import Optional


class ProjectType(str, Enum):
    web = "web"
    api = "api"
    cli = "cli"
    library = "library"


app = typer.Typer()

@app.command()
def init(
    name: Optional[str] = typer.Option(None, help="Project name (prompted if not given)"),
    project_type: Optional[ProjectType] = typer.Option(None, help="Project type"),
    author: Optional[str] = typer.Option(None, help="Author name"),
    confirm_create: bool = typer.Option(False, "--yes", "-y", help="Skip confirmation"),
):
    """Initialize a new project interactively."""
    # Prompt for missing values
    if name is None:
        name = typer.prompt("Project name")

    if project_type is None:
        type_str = typer.prompt(
            "Project type",
            type=typer.Choice([t.value for t in ProjectType], case_sensitive=False),
            default="web",
        )
        project_type = ProjectType(type_str)

    if author is None:
        author = typer.prompt("Author", default="Anonymous")

    # Display summary
    typer.echo("\n--- Project Summary ---")
    typer.secho(f"  Name:   {name}", fg=typer.colors.CYAN)
    typer.secho(f"  Type:   {project_type.value}", fg=typer.colors.CYAN)
    typer.secho(f"  Author: {author}", fg=typer.colors.CYAN)
    typer.echo("")

    # Confirm
    if not confirm_create:
        typer.confirm("Create this project?", abort=True)

    typer.secho(f"Project '{name}' created!", fg=typer.colors.GREEN, bold=True)


@app.command()
def setup_credentials():
    """Set up authentication credentials."""
    username = typer.prompt("Username")
    password = typer.prompt("Password", hide_input=True, confirmation_prompt=True)

    typer.echo(f"\nCredentials saved for user: {username}")
    typer.echo(f"Password length: {len(password)} characters")


if __name__ == "__main__":
    app()
```

### Example 6: Testing with CliRunner

```python
"""test_app.py -- Testing a Typer application with CliRunner."""
import pytest
from typer.testing import CliRunner

# Assume app is defined in myapp.py
from myapp import app

runner = CliRunner()


class TestGreetCommand:
    """Tests for the greet command."""

    def test_basic_greeting(self):
        result = runner.invoke(app, ["greet", "Alice"])
        assert result.exit_code == 0
        assert "Hello, Alice" in result.output

    def test_formal_greeting(self):
        result = runner.invoke(app, ["greet", "Alice", "--formal"])
        assert result.exit_code == 0
        assert "Good day, Alice" in result.output

    def test_missing_name(self):
        result = runner.invoke(app, ["greet"])
        assert result.exit_code != 0
        assert "Missing argument" in result.output

    def test_help_text(self):
        result = runner.invoke(app, ["greet", "--help"])
        assert result.exit_code == 0
        assert "Greet someone" in result.output


class TestDeployCommand:
    """Tests for the deploy command."""

    def test_deploy_default_env(self):
        result = runner.invoke(app, ["deploy", "api"])
        assert result.exit_code == 0
        assert "dev" in result.output

    def test_deploy_production_confirm(self):
        result = runner.invoke(
            app,
            ["deploy", "api", "--env", "production"],
            input="y\n",  # simulate typing "y" at the confirm prompt
        )
        assert result.exit_code == 0
        assert "production" in result.output

    def test_deploy_production_abort(self):
        result = runner.invoke(
            app,
            ["deploy", "api", "--env", "production"],
            input="n\n",  # simulate typing "n"
        )
        assert result.exit_code != 0  # Abort raises SystemExit(1)

    def test_invalid_environment(self):
        result = runner.invoke(app, ["deploy", "api", "--env", "invalid"])
        assert result.exit_code != 0
        assert "Invalid value" in result.output


class TestGlobalOptions:
    """Tests for app-level callback options."""

    def test_verbose_flag(self):
        result = runner.invoke(app, ["--verbose", "greet", "Alice"])
        assert result.exit_code == 0

    def test_dry_run(self):
        result = runner.invoke(app, ["--dry-run", "deploy", "api"])
        assert result.exit_code == 0
        assert "DRY RUN" in result.output


class TestEdgeCases:
    """Edge case and regression tests."""

    def test_empty_input(self):
        result = runner.invoke(app, [])
        # no_args_is_help=True means help is shown
        assert "Usage:" in result.output

    def test_exception_handling(self):
        result = runner.invoke(app, ["bad-command"])
        assert result.exit_code != 0

    def test_env_var_override(self):
        result = runner.invoke(
            app,
            ["config", "show"],
            env={"APP_ENV": "staging"},
        )
        assert result.exit_code == 0
```

### Example 7: Rich-Formatted Output

```python
"""richapp.py -- Rich integration with Typer."""
import typer
from typing import List, Optional
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.progress import Progress, SpinnerColumn, TextColumn
from rich import print as rprint
import time

app = typer.Typer(
    rich_markup_mode="rich",
    help="[bold blue]Rich CLI[/bold blue] -- A beautifully formatted CLI application.",
)
console = Console()


@app.command()
def status():
    """Show system [bold green]status[/bold green] in a formatted table."""
    table = Table(title="System Status", show_header=True, header_style="bold magenta")
    table.add_column("Service", style="cyan", width=20)
    table.add_column("Status", justify="center", width=12)
    table.add_column("Uptime", justify="right", style="green", width=15)
    table.add_column("Memory", justify="right", width=12)

    services = [
        ("API Server", "[green]Running[/green]", "14d 3h 22m", "256 MB"),
        ("Database", "[green]Running[/green]", "30d 1h 45m", "1.2 GB"),
        ("Cache", "[yellow]Degraded[/yellow]", "2d 8h 10m", "512 MB"),
        ("Worker", "[red]Stopped[/red]", "-", "0 MB"),
    ]
    for name, status, uptime, memory in services:
        table.add_row(name, status, uptime, memory)

    console.print(table)


@app.command()
def deploy(
    service: str = typer.Argument(..., help="Service to deploy"),
    version: str = typer.Option("latest", help="Version tag"),
):
    """Deploy a service with a [bold]progress indicator[/bold]."""
    console.print(Panel(
        f"Deploying [bold cyan]{service}[/bold cyan] version [bold]{version}[/bold]",
        title="Deployment",
        border_style="blue",
    ))

    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        transient=True,
    ) as progress:
        task = progress.add_task("Pulling image...", total=None)
        time.sleep(1)

        progress.update(task, description="Running tests...")
        time.sleep(1)

        progress.update(task, description="Deploying...")
        time.sleep(1)

        progress.update(task, description="Verifying health...")
        time.sleep(0.5)

    console.print(f"[bold green]Successfully deployed {service}:{version}[/bold green]")


@app.command()
def logs(
    service: str = typer.Argument(..., help="Service name"),
    lines: int = typer.Option(10, "--lines", "-n", help="Number of log lines"),
    level: Optional[str] = typer.Option(None, help="Filter by log level"),
):
    """Show recent [italic]logs[/italic] for a service."""
    console.print(f"[bold]Last {lines} logs for {service}:[/bold]\n")

    log_entries = [
        ("[dim]2025-01-15 10:23:01[/dim]", "[green]INFO[/green]", "Server started on port 8000"),
        ("[dim]2025-01-15 10:23:05[/dim]", "[green]INFO[/green]", "Connected to database"),
        ("[dim]2025-01-15 10:24:12[/dim]", "[yellow]WARN[/yellow]", "High memory usage: 85%"),
        ("[dim]2025-01-15 10:25:00[/dim]", "[green]INFO[/green]", "Health check passed"),
        ("[dim]2025-01-15 10:25:33[/dim]", "[red]ERROR[/red]", "Connection timeout to cache"),
    ]

    for timestamp, log_level, message in log_entries:
        if level is None or level.upper() in log_level.upper():
            console.print(f"  {timestamp} {log_level} {message}")


if __name__ == "__main__":
    app()
```

```bash
$ python richapp.py status
# Displays a beautifully formatted table with colors

$ python richapp.py deploy api --version v2.4.0
# Shows a panel and spinner progress

$ python richapp.py logs api --level error
# Filtered, color-coded log output
```

## Sources

- [Typer Official Documentation](https://typer.tiangolo.com/)
- [Typer GitHub Repository](https://github.com/fastapi/typer)
- [Typer on PyPI](https://pypi.org/project/typer/)
- [Click Documentation](https://click.palletsprojects.com/) (underlying library)
- [Rich Documentation](https://rich.readthedocs.io/) (optional integration)
