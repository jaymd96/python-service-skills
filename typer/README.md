# Typer -- Build CLI Applications with Python Type Hints

## Overview

**Typer** is a library for building command-line interface (CLI) applications in Python, created by Sebastian Ramirez (the author of FastAPI). It leverages Python type hints to define CLI parameters, producing robust CLIs with automatic help text, shell completion, and input validation -- all with minimal boilerplate. Under the hood, Typer is built on top of [Click](https://click.palletsprojects.com/), the most widely used Python CLI toolkit, and acts as a high-level, type-hint-driven wrapper around it.

### Key Features

- **Type hints as the source of truth**: Function parameter annotations define argument types, defaults, and help text.
- **Automatic `--help`**: Every command and sub-command gets auto-generated help pages.
- **Shell completion**: Built-in support for Bash, Zsh, Fish, and PowerShell tab completion.
- **Minimal code**: A simple CLI can be a single function with zero decorators.
- **Rich integration**: Optional pretty-printed output, formatted help pages, and styled error messages via [Rich](https://github.com/Textualize/rich).
- **Click-compatible**: Full access to the underlying Click infrastructure when needed.
- **Testable**: Ships with `CliRunner` for testing CLI applications without spawning subprocesses.

### Latest Stable Version

Typer **0.15.x** is the latest stable release line (as of early 2025). The package requires Python 3.7+ (3.8+ recommended). The version can be checked after installation:

```bash
python -c "import typer; print(typer.__version__)"
```

> **Note**: Check [PyPI](https://pypi.org/project/typer/) for the most current release.

---

## Installation

### Basic installation

```bash
pip install typer
```

This installs typer with its default dependencies (Click and typing-extensions).

### With Rich extra (recommended)

```bash
pip install "typer[all]"
```

This additionally installs:
- **rich** -- Beautiful formatted output, tracebacks, and help pages
- **shellingham** -- Automatic shell detection for completion installation

### Dependencies (installed automatically)

| Dependency | Purpose |
|---|---|
| `click` (>=8.0) | Underlying CLI framework |
| `typing-extensions` | Backported typing features |
| `rich` (optional, via `[all]`) | Styled terminal output |
| `shellingham` (optional, via `[all]`) | Shell detection |

---

## Core API

### `typer.run(func)` -- Simplest Single-Command CLI

The fastest way to turn a function into a CLI. This is ideal for scripts with a single command.

```python
import typer

def main(name: str, age: int = 25, formal: bool = False):
    """Greet someone by name."""
    greeting = "Good day" if formal else "Hello"
    typer.echo(f"{greeting}, {name}! You are {age} years old.")

if __name__ == "__main__":
    typer.run(main)
```

```bash
$ python greet.py --help
Usage: greet.py [OPTIONS] NAME

  Greet someone by name.

Options:
  --age INTEGER   [default: 25]
  --formal / --no-formal  [default: no-formal]
  --help          Show this message and exit.

$ python greet.py Alice --age 30 --formal
Good day, Alice! You are 30 years old.
```

**How it works**: `typer.run()` creates a `Typer` app internally, registers the function as the sole command, and invokes it. It is equivalent to:

```python
app = typer.Typer()
app.command()(main)
app()
```

### `app = typer.Typer()` -- Creating Multi-Command Apps

For applications with multiple commands (or when you need more control), create a `Typer` instance explicitly.

```python
import typer

app = typer.Typer(help="A tool for managing users.")

@app.command()
def create(name: str):
    """Create a new user."""
    typer.echo(f"Created user: {name}")

@app.command()
def delete(name: str, force: bool = False):
    """Delete a user."""
    if force or typer.confirm(f"Delete {name}?"):
        typer.echo(f"Deleted user: {name}")

if __name__ == "__main__":
    app()
```

#### `typer.Typer()` Constructor Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `str \| None` | `None` | Name shown in help text |
| `help` | `str \| None` | `None` | Help text for the app/group |
| `add_completion` | `bool` | `True` | Add `--install-completion` / `--show-completion` |
| `no_args_is_help` | `bool` | `False` | Show help when called with no arguments |
| `invoke_without_command` | `bool` | `False` | Run callback even without a subcommand |
| `result_callback` | `callable \| None` | `None` | Called with the return value of the invoked command |
| `rich_markup_mode` | `str \| None` | `None` | `"rich"`, `"markdown"`, or `None` for help text formatting |
| `pretty_exceptions_enable` | `bool` | `True` | Use Rich for exception display |
| `pretty_exceptions_show_locals` | `bool` | `True` | Show local variables in tracebacks |
| `pretty_exceptions_short` | `bool` | `True` | Use short tracebacks |

### `@app.command()` -- Registering Commands

Each decorated function becomes a CLI subcommand. The function name (converted from `snake_case` to `kebab-case`) becomes the command name.

```python
app = typer.Typer()

@app.command()
def run_server(port: int = 8000):
    """Start the development server."""
    typer.echo(f"Server running on port {port}")

@app.command()
def run_worker(queues: str = "default"):
    """Start a background worker."""
    typer.echo(f"Worker processing queues: {queues}")
```

```bash
$ python app.py --help
Usage: app.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  run-server  Start the development server.
  run-worker  Start a background worker.
```

#### `@app.command()` Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `str \| None` | `None` | Override the command name |
| `help` | `str \| None` | `None` | Override help text (default: docstring) |
| `epilog` | `str \| None` | `None` | Text shown after help |
| `short_help` | `str \| None` | `None` | Short help shown in parent group listing |
| `hidden` | `bool` | `False` | Hide from help output |
| `deprecated` | `bool` | `False` | Mark command as deprecated |
| `rich_help_panel` | `str \| None` | `None` | Group the command under a Rich help panel |

### `@app.callback()` -- App-Level Options and Help

The callback defines options that apply to the app itself (before any subcommand) and sets the app-level help text.

```python
app = typer.Typer()

@app.callback()
def main(
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Enable verbose output"),
    config: str = typer.Option("config.yaml", help="Path to config file"),
):
    """
    My CLI Application.

    Manage deployments and configurations.
    """
    if verbose:
        typer.echo("Verbose mode enabled")

@app.command()
def deploy(env: str):
    """Deploy to an environment."""
    typer.echo(f"Deploying to {env}")
```

```bash
$ python app.py --verbose deploy production
Verbose mode enabled
Deploying to production
```

> **Important**: Set `invoke_without_command=True` on the `Typer()` instance if you want the callback to execute even when no subcommand is provided.

### `typer.Argument(...)` -- Positional Arguments with Metadata

Positional arguments are parameters without `--` prefixes. Use `typer.Argument()` to add help text, defaults, and validation.

```python
import typer
from typing import Optional

def main(
    name: str = typer.Argument(..., help="The user's name"),
    greeting: str = typer.Argument("Hello", help="The greeting to use"),
):
    typer.echo(f"{greeting}, {name}!")
```

#### `typer.Argument()` Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `default` | `any` | (required: `...`) | Default value. Use `...` (Ellipsis) for required. |
| `help` | `str \| None` | `None` | Help text shown in `--help` |
| `metavar` | `str \| None` | `None` | Display name in usage line |
| `hidden` | `bool` | `False` | Hide from help output |
| `envvar` | `str \| None` | `None` | Environment variable fallback |
| `show_default` | `bool` | `True` | Show default value in help |
| `show_envvar` | `bool` | `True` | Show env var name in help |
| `min` | `int \| float \| None` | `None` | Minimum value (numeric types) |
| `max` | `int \| float \| None` | `None` | Maximum value (numeric types) |
| `clamp` | `bool` | `False` | Clamp value to min/max instead of erroring |
| `exists` | `bool` | `False` | For `Path`: require path to exist |
| `file_okay` | `bool` | `True` | For `Path`: allow files |
| `dir_okay` | `bool` | `True` | For `Path`: allow directories |
| `rich_help_panel` | `str \| None` | `None` | Group under a Rich panel in help |
| `callback` | `callable \| None` | `None` | Validation/transformation callback |
| `autocompletion` | `callable \| None` | `None` | Shell completion callback |

### `typer.Option(...)` -- Optional Flags with Metadata

Options are named CLI parameters prefixed with `--`. By default, typer converts parameter names to `--kebab-case` flags.

```python
import typer
from pathlib import Path

def main(
    name: str = typer.Option(..., "--name", "-n", help="Your name"),
    output: Path = typer.Option("./out", help="Output directory"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Verbose mode"),
    count: int = typer.Option(1, min=1, max=10, help="Repeat count"),
):
    typer.echo(f"Name: {name}, Output: {output}, Verbose: {verbose}, Count: {count}")
```

#### `typer.Option()` Parameters

All parameters from `typer.Argument()` apply, plus:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `*param_decls` | `str` | auto-generated | Custom flag names (e.g., `"--name"`, `"-n"`) |
| `prompt` | `bool \| str` | `False` | Prompt the user for input if not provided. `True` uses param name; a string provides custom prompt text. |
| `confirmation_prompt` | `bool` | `False` | Ask for confirmation (e.g., passwords) |
| `hide_input` | `bool` | `False` | Hide typed characters (for passwords) |
| `is_eager` | `bool` | `False` | Process before other options (useful for `--version`) |

### Type Annotations and CLI Types

Typer maps Python type annotations to Click parameter types automatically.

| Python Type | CLI Behavior | Example Input |
|---|---|---|
| `str` | Plain string | `hello` |
| `int` | Integer with validation | `42` |
| `float` | Float with validation | `3.14` |
| `bool` | Flag: `--flag` / `--no-flag` | `--verbose` |
| `pathlib.Path` | File/directory path (with optional existence check) | `./data.csv` |
| `enum.Enum` | Choice from enum values | `debug` |
| `datetime.datetime` | Date/time parsing | `2025-01-15` |
| `typing.Optional[X]` | Optional parameter (can be `None`) | (omitted) |
| `typing.List[X]` | Multiple values | `--item a --item b` |
| `typing.Tuple[X, Y]` | Fixed-length multi-value | `--point 1 2` |
| `uuid.UUID` | UUID string | `550e8400-e29b-...` |

#### Enum Example (Choices)

```python
import typer
from enum import Enum

class Color(str, Enum):
    red = "red"
    green = "green"
    blue = "blue"

def main(color: Color = Color.red):
    typer.echo(f"Selected: {color.value}")

typer.run(main)
```

```bash
$ python app.py --color green
Selected: green

$ python app.py --color purple
Usage: app.py [OPTIONS]
Error: Invalid value for '--color': 'purple' is not one of 'red', 'green', 'blue'.
```

#### List Example (Multiple Values)

```python
from typing import List, Optional
import typer

def main(users: Optional[List[str]] = typer.Option(None, help="Users to notify")):
    if users:
        for user in users:
            typer.echo(f"Notifying: {user}")
    else:
        typer.echo("No users specified")

typer.run(main)
```

```bash
$ python app.py --users alice --users bob
Notifying: alice
Notifying: bob
```

### Output Helpers

#### `typer.echo(message, err=False, color=None)`

Prints a message to stdout (or stderr when `err=True`). Works consistently across platforms including when output is piped.

```python
typer.echo("Normal output")
typer.echo("Error message", err=True)   # writes to stderr
typer.echo("No newline", nl=False)       # suppress trailing newline
```

#### `typer.secho(message, fg=None, bg=None, bold=None, ...)`

Styled echo -- combines `echo` with ANSI color/style formatting (a Click feature).

```python
typer.secho("Success!", fg=typer.colors.GREEN, bold=True)
typer.secho("Warning!", fg=typer.colors.YELLOW)
typer.secho("Error!", fg=typer.colors.RED, err=True)
```

Available `typer.colors`: `BLACK`, `RED`, `GREEN`, `YELLOW`, `BLUE`, `MAGENTA`, `CYAN`, `WHITE`, `BRIGHT_BLACK`, `BRIGHT_RED`, etc.

#### `typer.style(text, fg=None, bg=None, bold=None, ...)`

Returns a styled string without printing it.

```python
styled = typer.style("important", fg=typer.colors.RED, bold=True)
typer.echo(f"This is {styled} text")
```

### Interactive Input

#### `typer.prompt(text, default=None, type=None, hide_input=False)`

Prompts the user for input, with optional type conversion and default value.

```python
name = typer.prompt("What is your name")
age = typer.prompt("How old are you", type=int, default=25)
password = typer.prompt("Password", hide_input=True, confirmation_prompt=True)
```

#### `typer.confirm(text, default=False, abort=False)`

Asks a yes/no question. If `abort=True`, answering "no" raises `typer.Abort()`.

```python
delete = typer.confirm("Are you sure you want to delete?")
if delete:
    typer.echo("Deleted!")

# Or auto-abort on "no":
typer.confirm("Destroy everything?", abort=True)
typer.echo("Destroyed!")  # only reached if user said yes
```

### Flow Control

#### `typer.Exit(code=0)`

Raise to exit the CLI with a specific exit code. This is a clean exit (not an error).

```python
def main(name: str):
    if name == "root":
        typer.echo("Access denied")
        raise typer.Exit(code=1)
    typer.echo(f"Hello, {name}")
```

#### `typer.Abort()`

Raise to abort the CLI with an "Aborted!" message. Conventionally signals user cancellation.

```python
def main():
    typer.confirm("Continue?", abort=True)  # raises Abort if "no"
    typer.echo("Continuing...")
```

### `typer.Context` -- Accessing Click Context

Use `typer.Context` as a parameter type to access the underlying Click context object. This gives you access to invoked subcommand info, parent context, and more.

```python
import typer

app = typer.Typer()

@app.callback(invoke_without_command=True)
def main(ctx: typer.Context, verbose: bool = False):
    """My CLI app."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose
    if ctx.invoked_subcommand is None:
        typer.echo("No command provided. Use --help.")

@app.command()
def greet(ctx: typer.Context, name: str):
    """Say hello."""
    if ctx.obj.get("verbose"):
        typer.echo(f"[DEBUG] Greeting {name}")
    typer.echo(f"Hello, {name}!")
```

#### Useful `Context` Attributes

| Attribute | Description |
|---|---|
| `ctx.invoked_subcommand` | Name of the subcommand about to be called (or `None`) |
| `ctx.obj` | Shared object dict for passing data between callback and commands |
| `ctx.parent` | Parent context (for nested groups) |
| `ctx.params` | Dict of all parameters for the current command |
| `ctx.command` | The Click `Command` object |
| `ctx.info_name` | The name of the script/command |

---

## Sub-commands and Groups

### `app.add_typer(sub_app, name=...)` -- Nesting Command Groups

Build hierarchical CLIs by nesting `Typer` instances. Each nested `Typer` becomes a command group.

```python
import typer

# Root app
app = typer.Typer(help="Deployment manager")

# Sub-app for user commands
users_app = typer.Typer(help="Manage users")
app.add_typer(users_app, name="users")

# Sub-app for config commands
config_app = typer.Typer(help="Manage configuration")
app.add_typer(config_app, name="config")

@users_app.command()
def create(name: str, role: str = "viewer"):
    """Create a new user."""
    typer.echo(f"Created user {name} with role {role}")

@users_app.command("list")
def list_users():
    """List all users."""
    typer.echo("alice, bob, charlie")

@config_app.command()
def show():
    """Show current configuration."""
    typer.echo("config: default")

@config_app.command("set")
def set_config(key: str, value: str):
    """Set a configuration value."""
    typer.echo(f"Set {key} = {value}")

if __name__ == "__main__":
    app()
```

```bash
$ python app.py --help
Usage: app.py [OPTIONS] COMMAND [ARGS]...

  Deployment manager

Commands:
  config  Manage configuration
  users   Manage users

$ python app.py users create Alice --role admin
Created user Alice with role admin

$ python app.py config set theme dark
Set theme = dark
```

### Shared Options via Callback

Use `@sub_app.callback()` to define options shared by all commands in a group.

```python
users_app = typer.Typer()

@users_app.callback()
def users_callback(
    ctx: typer.Context,
    db_url: str = typer.Option("sqlite:///users.db", help="Database URL"),
):
    """Manage users in the system."""
    ctx.ensure_object(dict)
    ctx.obj["db_url"] = db_url

@users_app.command()
def create(ctx: typer.Context, name: str):
    db_url = ctx.obj["db_url"]
    typer.echo(f"Creating {name} in {db_url}")
```

### Deep Nesting

Groups can be nested arbitrarily deep:

```python
app = typer.Typer()
cloud_app = typer.Typer(help="Cloud operations")
aws_app = typer.Typer(help="AWS-specific commands")

app.add_typer(cloud_app, name="cloud")
cloud_app.add_typer(aws_app, name="aws")

@aws_app.command()
def deploy(region: str = "us-east-1"):
    typer.echo(f"Deploying to AWS {region}")
```

```bash
$ python app.py cloud aws deploy --region eu-west-1
Deploying to AWS eu-west-1
```

---

## Autocompletion -- Shell Completion Support

Typer provides built-in shell completion for Bash, Zsh, Fish, and PowerShell.

### Installing Completion

```bash
# Auto-detect shell
$ python app.py --install-completion

# For a specific shell
$ python app.py --install-completion bash
$ python app.py --install-completion zsh
$ python app.py --install-completion fish
```

### Showing Completion Script

```bash
$ python app.py --show-completion
```

### Custom Autocompletion

Provide dynamic completions for arguments and options:

```python
import typer

def complete_name(incomplete: str) -> list[str]:
    names = ["alice", "bob", "charlie", "diana"]
    return [n for n in names if n.startswith(incomplete)]

app = typer.Typer()

@app.command()
def greet(name: str = typer.Argument(..., autocompletion=complete_name)):
    typer.echo(f"Hello, {name}")
```

For richer completions (with help text), yield tuples:

```python
def complete_name(incomplete: str):
    names = [
        ("alice", "Engineering lead"),
        ("bob", "Product manager"),
        ("charlie", "Designer"),
    ]
    for name, help_text in names:
        if name.startswith(incomplete):
            yield name, help_text
```

### Disabling Completion

```python
app = typer.Typer(add_completion=False)
```

---

## Testing -- `typer.testing.CliRunner`

Typer includes `CliRunner` (wrapping Click's `CliRunner`) for testing CLI applications without spawning subprocesses.

### Basic Test Pattern

```python
from typer.testing import CliRunner
from myapp import app

runner = CliRunner()

def test_greet():
    result = runner.invoke(app, ["Alice", "--formal"])
    assert result.exit_code == 0
    assert "Good day, Alice" in result.output

def test_help():
    result = runner.invoke(app, ["--help"])
    assert result.exit_code == 0
    assert "Usage:" in result.output

def test_invalid_input():
    result = runner.invoke(app, ["--age", "not-a-number"])
    assert result.exit_code != 0
    assert "Invalid value" in result.output
```

### `CliRunner` Methods and Result Object

```python
runner = CliRunner(
    mix_stderr=False,    # Separate stdout and stderr
    env={"MY_VAR": "1"}, # Set environment variables
)

result = runner.invoke(
    app,
    ["command", "--option", "value"],  # CLI arguments
    input="yes\n",                      # Simulated stdin (for prompts)
    env={"KEY": "value"},               # Override env vars for this call
    catch_exceptions=False,             # Let exceptions propagate (useful for debugging)
)
```

#### `Result` Object Attributes

| Attribute | Type | Description |
|---|---|---|
| `result.exit_code` | `int` | Exit code (0 = success) |
| `result.output` | `str` | Captured stdout |
| `result.stderr` | `str` | Captured stderr (when `mix_stderr=False`) |
| `result.exception` | `Exception \| None` | Exception if one was raised |
| `result.runner` | `CliRunner` | The runner that produced this result |

### Testing with Prompts

```python
def test_confirm_yes():
    result = runner.invoke(app, ["delete", "myfile"], input="y\n")
    assert result.exit_code == 0
    assert "Deleted" in result.output

def test_confirm_no():
    result = runner.invoke(app, ["delete", "myfile"], input="n\n")
    assert "Aborted" in result.output
```

### Testing with Environment Variables

```python
def test_with_env():
    result = runner.invoke(app, ["deploy"], env={"DEPLOY_TARGET": "staging"})
    assert "staging" in result.output
```

---

## Rich Integration

When installed with `typer[all]` (or with `rich` available), Typer automatically provides enhanced output.

### Rich Help Formatting

Enable Rich markup or Markdown in help text:

```python
app = typer.Typer(rich_markup_mode="rich")

@app.command()
def deploy(
    env: str = typer.Argument(..., help="Target [bold green]environment[/bold green]"),
    dry_run: bool = typer.Option(False, help="Simulate without [italic]making changes[/italic]"),
):
    """
    Deploy the application.

    This will [bold]build[/bold], [bold]test[/bold], and [bold]push[/bold] the code.
    """
    typer.echo(f"Deploying to {env}")
```

Or use Markdown mode:

```python
app = typer.Typer(rich_markup_mode="markdown")

@app.command()
def deploy(
    env: str = typer.Argument(..., help="Target **environment**"),
):
    """
    Deploy the application.

    This will **build**, **test**, and **push** the code.
    """
```

### Rich Help Panels

Group parameters and commands into named panels:

```python
@app.command()
def process(
    input_file: Path = typer.Argument(..., help="Input file", rich_help_panel="Input"),
    output_file: Path = typer.Option("out.txt", help="Output file", rich_help_panel="Output"),
    format: str = typer.Option("json", help="Output format", rich_help_panel="Output"),
    verbose: bool = typer.Option(False, help="Verbose logging", rich_help_panel="Debug"),
):
    """Process data files."""
    ...
```

### Pretty Exceptions

Typer automatically uses Rich for displaying tracebacks (when Rich is installed). Configure via:

```python
app = typer.Typer(
    pretty_exceptions_enable=True,       # Enable Rich tracebacks (default)
    pretty_exceptions_show_locals=True,   # Show local variables (default)
    pretty_exceptions_short=True,         # Short tracebacks (default)
)
```

### Using Rich Directly with Typer

You can use Rich's `Console` alongside Typer for advanced output:

```python
import typer
from rich.console import Console
from rich.table import Table
from rich.progress import track

app = typer.Typer()
console = Console()

@app.command()
def show_users():
    """Display users in a formatted table."""
    table = Table(title="Users")
    table.add_column("Name", style="cyan")
    table.add_column("Role", style="green")
    table.add_column("Active", style="magenta")

    table.add_row("Alice", "Admin", "Yes")
    table.add_row("Bob", "Editor", "Yes")
    table.add_row("Charlie", "Viewer", "No")

    console.print(table)

@app.command()
def process(count: int = 100):
    """Process items with a progress bar."""
    for _ in track(range(count), description="Processing..."):
        pass  # actual work here
    console.print("[bold green]Done![/bold green]")
```

---

## Click Interop -- Accessing the Underlying Click Layer

Since Typer is built on Click, you can drop down to Click when you need features Typer does not directly expose.

### Getting the Click Command Object

```python
import click
import typer

app = typer.Typer()

@app.command()
def main(name: str):
    typer.echo(f"Hello, {name}")

# Convert Typer app to Click command (useful for plugins, entry points)
click_app = typer.main.get_command(app)
```

### Using Click Decorators Alongside Typer

```python
import click
import typer

app = typer.Typer()

@app.command()
@click.pass_context
def main(ctx: click.Context, name: str):
    """Access Click context directly."""
    typer.echo(f"Command: {ctx.info_name}")
    typer.echo(f"Hello, {name}")
```

### Using Click Types Not Wrapped by Typer

```python
import click
import typer
from typing import Annotated

app = typer.Typer()

@app.command()
def main(
    color: Annotated[str, typer.Option(click_type=click.Choice(["red", "green", "blue"], case_sensitive=False))] = "red",
):
    typer.echo(f"Color: {color}")
```

### Registering Click Commands in a Typer App

```python
import click
import typer

app = typer.Typer()

@click.command()
@click.option("--name", prompt="Your name")
def legacy_hello(name: str):
    click.echo(f"Hello from Click, {name}!")

# Register the Click command with Typer
app.command(name="legacy")(legacy_hello)
# Or use: typer.main.get_command(app).add_command(legacy_hello, "legacy")
```

### Entry Points (setup.py / pyproject.toml)

```toml
# pyproject.toml
[project.scripts]
mycli = "mypackage.cli:app"
```

For this to work, the `app` object must be callable. Typer apps are callable by default (`app()` invokes the CLI).

---

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

---

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

---

## Version Callback Pattern

A very common pattern is adding a `--version` flag:

```python
from importlib.metadata import version as get_version

app = typer.Typer()

def version_callback(value: bool):
    if value:
        ver = get_version("mypackage")
        typer.echo(f"mypackage v{ver}")
        raise typer.Exit()

@app.callback()
def main(
    version: bool = typer.Option(
        False,
        "--version",
        "-V",
        help="Show version and exit",
        callback=version_callback,
        is_eager=True,
    ),
):
    """My CLI application."""
    pass
```

The `is_eager=True` ensures `--version` is processed before other parameters, so `myapp --version` works even if required arguments are missing.

---

## Annotated Syntax (Python 3.9+)

Typer supports the `Annotated` syntax for cleaner type hints (recommended for new code):

```python
from typing import Annotated
import typer

app = typer.Typer()

@app.command()
def main(
    name: Annotated[str, typer.Argument(help="Your name")],
    age: Annotated[int, typer.Option(help="Your age")] = 25,
    verbose: Annotated[bool, typer.Option("--verbose", "-v")] = False,
):
    """Greet with Annotated syntax."""
    typer.echo(f"Hello, {name}! Age: {age}")
```

This is equivalent to the default-value syntax but keeps the type hint clean and separates the metadata.

---

## Sources

- [Typer Official Documentation](https://typer.tiangolo.com/)
- [Typer GitHub Repository](https://github.com/fastapi/typer)
- [Typer on PyPI](https://pypi.org/project/typer/)
- [Click Documentation](https://click.palletsprojects.com/) (underlying library)
- [Rich Documentation](https://rich.readthedocs.io/) (optional integration)
