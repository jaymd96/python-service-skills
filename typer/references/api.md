# Typer — API Reference

> Part of the typer skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API](#core-api)
  - [typer.run(func) -- Simplest Single-Command CLI](#typerrunfunc----simplest-single-command-cli)
  - [app = typer.Typer() -- Creating Multi-Command Apps](#app--typertyper----creating-multi-command-apps)
  - [@app.command() -- Registering Commands](#appcommand----registering-commands)
  - [@app.callback() -- App-Level Options and Help](#appcallback----app-level-options-and-help)
  - [typer.Argument(...) -- Positional Arguments with Metadata](#typerargument----positional-arguments-with-metadata)
  - [typer.Option(...) -- Optional Flags with Metadata](#typeroption----optional-flags-with-metadata)
  - [Type Annotations and CLI Types](#type-annotations-and-cli-types)
  - [Output Helpers](#output-helpers)
  - [Interactive Input](#interactive-input)
  - [Flow Control](#flow-control)
  - [typer.Context -- Accessing Click Context](#typercontext----accessing-click-context)
- [Annotated Syntax (Python 3.9+)](#annotated-syntax-python-39)

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
