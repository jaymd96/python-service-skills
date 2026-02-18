# Click — API Reference

> Part of the click skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [@click.command() -- Defining Commands](#clickcommand----defining-commands)
- [@click.option() -- Adding Options](#clickoption----adding-options)
- [@click.argument() -- Adding Arguments](#clickargument----adding-arguments)
- [Parameter Types](#parameter-types)
- [Callbacks and Value Processing](#callbacks-and-value-processing)
- [Output Functions](#output-functions)
- [Summary Reference Table](#summary-reference-table)

---

## `@click.command()` -- Defining Commands

The `@click.command()` decorator converts a Python function into a Click command. The function's docstring becomes the help text.

```python
import click

@click.command()
def hello():
    """Simple program that greets the world."""
    click.echo("Hello, World!")

if __name__ == "__main__":
    hello()
```

Running this:

```
$ python hello.py
Hello, World!

$ python hello.py --help
Usage: hello.py [OPTIONS]

  Simple program that greets the world.

Options:
  --help  Show this message and exit.
```

### `@click.command()` Parameters

```python
@click.command(
    name="my-cmd",              # Override command name (default: function name)
    cls=click.Command,          # Custom Command class
    context_settings=dict(),    # Dict passed to the Context constructor
    help="Override help text",  # Override docstring
    epilog="Shown after help",  # Text shown after the help body
    short_help="Brief text",    # Short help for parent group listings
    add_help_option=True,       # Whether to add --help (default True)
    no_args_is_help=False,      # If True, show help when invoked with no args
    hidden=False,               # If True, hide from help output
    deprecated=False,           # If True, show deprecation warning
)
def my_command():
    ...
```

---

## `@click.option()` -- Adding Options

Options are keyword-based parameters prefixed with `--` (and optionally `-` for short form).

```python
@click.command()
@click.option("--name", "-n", default="World", help="Who to greet.")
@click.option("--count", "-c", default=1, type=int, help="Number of greetings.")
def hello(name, count):
    """Greet someone COUNT times."""
    for _ in range(count):
        click.echo(f"Hello, {name}!")
```

```
$ python hello.py --name Alice --count 3
Hello, Alice!
Hello, Alice!
Hello, Alice!

$ python hello.py -n Bob -c 2
Hello, Bob!
Hello, Bob!
```

### Full `@click.option()` Parameter Reference

```python
@click.option(
    "--name", "-n",           # Long and short parameter declarations
    default=None,             # Default value (also infers type if not set)
    type=str,                 # Parameter type (see Types section)
    required=False,           # Whether the option is required
    help="Description",       # Help text shown in --help
    show_default=False,       # Whether to show the default in help text
    prompt=False,             # If True or a string, prompt for input if not provided
    confirmation_prompt=False,# Prompt twice for confirmation (for passwords)
    hide_input=False,         # Hide typed input (for passwords)
    is_flag=False,            # Treat as a boolean flag
    flag_value=None,          # Value when flag is set
    count=False,              # Count the number of times option is used (-vvv)
    multiple=False,           # Allow option to be provided multiple times
    envvar=None,              # Environment variable name(s) to read from
    shell_complete=None,      # Shell completion callback
    callback=None,            # Callback function for value processing
    nargs=1,                  # Number of values the option takes
    metavar=None,             # Display name in help text
    expose_value=True,        # Whether to pass the value to the command function
    is_eager=False,           # Process before other parameters
    show_envvar=False,        # Show envvar name in help text
)
```

### Common Option Patterns

**Required option:**

```python
@click.option("--name", required=True, help="Your name.")
```

**Option with prompt:**

```python
@click.option("--password", prompt=True, hide_input=True, confirmation_prompt=True)
```

**Boolean flags:**

```python
@click.option("--verbose/--no-verbose", default=False, help="Enable verbose output.")
# Usage: --verbose or --no-verbose

@click.option("--debug", is_flag=True, help="Enable debug mode.")
# Usage: --debug (sets to True)
```

**Counting flags (verbosity):**

```python
@click.option("-v", "--verbose", count=True, help="Increase verbosity.")
# Usage: -vvv sets verbose=3
```

**Multiple values:**

```python
@click.option("--include", "-I", multiple=True, help="Paths to include.")
# Usage: --include foo --include bar => include=("foo", "bar")
```

**Multi-value option (tuple):**

```python
@click.option("--pos", nargs=2, type=float, help="X Y coordinates.")
# Usage: --pos 1.0 2.5 => pos=(1.0, 2.5)
```

**Environment variable fallback:**

```python
@click.option("--api-key", envvar="MY_API_KEY", help="API key.")
# Reads from --api-key flag first, then MY_API_KEY env var
```

**Choice option:**

```python
@click.option("--format", type=click.Choice(["json", "csv", "table"], case_sensitive=False))
```

**Range option:**

```python
@click.option("--count", type=click.IntRange(1, 100), default=10)
@click.option("--rate", type=click.FloatRange(0.0, 1.0), default=0.5)
```

---

## `@click.argument()` -- Adding Arguments

Arguments are positional parameters (without `--` prefix). They are typically used for required inputs.

```python
@click.command()
@click.argument("filename")
def process(filename):
    """Process the given FILENAME."""
    click.echo(f"Processing {filename}...")
```

```
$ python cli.py report.csv
Processing report.csv...
```

### `@click.argument()` Parameter Reference

```python
@click.argument(
    "name",                   # Parameter name (also used as the variable name)
    default=None,             # Default value (makes the argument optional)
    type=str,                 # Parameter type
    required=True,            # Whether the argument is required (default True)
    nargs=1,                  # Number of values (-1 for unlimited)
    envvar=None,              # Environment variable fallback
    callback=None,            # Value processing callback
    metavar=None,             # Display name in help
    expose_value=True,        # Whether to pass to the function
    is_eager=False,           # Process before other parameters
    shell_complete=None,      # Shell completion callback
)
```

### Variadic Arguments

Use `nargs=-1` to accept an unlimited number of values:

```python
@click.command()
@click.argument("files", nargs=-1, required=True)
def cat(files):
    """Concatenate and display FILES."""
    for filename in files:
        with open(filename) as f:
            click.echo(f.read(), nl=False)
```

```
$ python cli.py file1.txt file2.txt file3.txt
```

The variadic argument collects all remaining positional values into a tuple. Only the last argument can be variadic.

### Optional Arguments

```python
@click.command()
@click.argument("output", default="-")  # "-" is a common convention for stdout
def export(output):
    """Export data to OUTPUT file (default: stdout)."""
    if output == "-":
        click.echo("Writing to stdout")
    else:
        click.echo(f"Writing to {output}")
```

### File Arguments

```python
@click.command()
@click.argument("input", type=click.File("r"))
@click.argument("output", type=click.File("w"), default="-")
def transform(input, output):
    """Read from INPUT and write to OUTPUT."""
    for line in input:
        output.write(line.upper())
```

`click.File` handles opening, closing, and the special filename `-` (stdin/stdout) automatically.

---

## Parameter Types

Click has a rich type system for validating and converting parameter values.

### Built-in Types

| Type | Description | Example |
|------|-------------|---------|
| `click.STRING` | Default. Any string. | `--name foo` |
| `click.INT` | Integer. | `--count 42` |
| `click.FLOAT` | Floating-point number. | `--rate 3.14` |
| `click.BOOL` | Boolean. Accepts `true/false`, `yes/no`, `1/0`. | `--flag true` |
| `click.UUID` | UUID string, converted to `uuid.UUID`. | `--id 12345678-...` |
| `click.UNPROCESSED` | Raw string, no processing. | -- |

### Parameterized Types

| Type | Description | Example |
|------|-------------|---------|
| `click.Choice(choices, case_sensitive=True)` | Must be one of the listed values. | `Choice(["a", "b", "c"])` |
| `click.IntRange(min, max, min_open, max_open, clamp)` | Integer within a range. | `IntRange(0, 100)` |
| `click.FloatRange(min, max, min_open, max_open, clamp)` | Float within a range. | `FloatRange(0.0, 1.0)` |
| `click.DateTime(formats)` | Parses date/time strings. | `DateTime(["%Y-%m-%d"])` |
| `click.Path(exists, file_okay, dir_okay, readable, writable, resolve_path, path_type)` | Filesystem path with validation. | `Path(exists=True)` |
| `click.File(mode, encoding, errors, lazy, atomic)` | Opens a file handle. Supports `-` for stdin/stdout. | `File("r")` |
| `click.Tuple(types)` | Multiple values of specified types. | `Tuple([int, str])` |

### `click.Path` in Detail

```python
@click.option(
    "--config",
    type=click.Path(
        exists=True,         # Path must exist
        file_okay=True,      # Files are allowed
        dir_okay=False,      # Directories are not allowed
        readable=True,       # Must be readable
        writable=False,      # No write check
        resolve_path=True,   # Resolve to absolute path
        path_type=None,      # Convert to this type (e.g., pathlib.Path)
    ),
)
def load(config):
    ...
```

Using `pathlib.Path`:

```python
from pathlib import Path

@click.option("--output", type=click.Path(path_type=Path))
def export(output: Path):
    output.write_text("data")
```

### `click.DateTime`

```python
@click.option("--start", type=click.DateTime(formats=["%Y-%m-%d", "%Y-%m-%dT%H:%M:%S"]))
def report(start):
    click.echo(f"Report starting from {start}")
```

```
$ python cli.py --start 2025-01-15
Report starting from 2025-01-15 00:00:00
```

### Custom Types

Create custom types by subclassing `click.ParamType`:

```python
class CommaSeparated(click.ParamType):
    """A type that splits a comma-separated string into a list."""
    name = "comma_separated"

    def convert(self, value, param, ctx):
        if isinstance(value, list):
            return value
        try:
            return [item.strip() for item in value.split(",")]
        except AttributeError:
            self.fail(f"{value!r} is not a valid comma-separated list", param, ctx)

COMMA_SEPARATED = CommaSeparated()

@click.command()
@click.option("--tags", type=COMMA_SEPARATED, default="", help="Comma-separated tags.")
def search(tags):
    for tag in tags:
        click.echo(f"Tag: {tag}")
```

```
$ python cli.py --tags "python,cli,click"
Tag: python
Tag: cli
Tag: click
```

---

## Callbacks and Value Processing

### Option Callbacks

Callbacks let you transform or validate values as they are parsed:

```python
def validate_positive(ctx, param, value):
    if value is not None and value <= 0:
        raise click.BadParameter("Must be a positive number.")
    return value

@click.command()
@click.option("--count", type=int, callback=validate_positive)
def repeat(count):
    click.echo(f"Count: {count}")
```

The callback signature is `(ctx, param, value) -> value`:
- `ctx` -- the `click.Context` instance
- `param` -- the `click.Parameter` instance
- `value` -- the parsed value

### Eager Options

Eager options are processed before all other parameters. This is the standard pattern for `--version`:

```python
def print_version(ctx, param, value):
    if not value or ctx.resilient_parsing:
        return
    click.echo("Version 1.0.0")
    ctx.exit()

@click.command()
@click.option(
    "--version",
    is_flag=True,
    callback=print_version,
    expose_value=False,
    is_eager=True,
    help="Show version and exit.",
)
def cli():
    click.echo("Running the CLI...")
```

Click also provides a built-in shorthand:

```python
@click.command()
@click.version_option(version="1.0.0", prog_name="my-tool")
def cli():
    ...
```

---

## Output Functions

### `click.echo()`

The primary output function. Preferred over `print()` because it handles encoding issues, supports color, and works correctly across platforms including broken pipe scenarios.

```python
click.echo("Hello, World!")                      # Print to stdout
click.echo("Error!", err=True)                   # Print to stderr
click.echo("No newline", nl=False)               # No trailing newline
click.echo(click.style("Bold!", bold=True))      # Styled output
click.echo(click.style("Error!", fg="red"), err=True)  # Colored stderr
```

### `click.style()` and `click.secho()`

```python
# Style a string (returns styled string, does not print)
styled = click.style("Warning", fg="yellow", bold=True)

# secho = style + echo in one call
click.secho("Success!", fg="green", bold=True)
click.secho("Error!", fg="red", err=True)
click.secho("Info", fg="blue", underline=True)
```

Available style parameters:
- `fg` -- foreground color: `"black"`, `"red"`, `"green"`, `"yellow"`, `"blue"`, `"magenta"`, `"cyan"`, `"white"`, `"bright_black"`, `"bright_red"`, etc., or an integer 0-255, or an `(r, g, b)` tuple
- `bg` -- background color (same values as `fg`)
- `bold`, `dim`, `underline`, `overline`, `italic`, `blink`, `reverse`, `strikethrough` -- boolean style flags

### `click.prompt()`

Interactive input with type validation:

```python
name = click.prompt("What is your name")
# What is your name: <user types>

age = click.prompt("How old are you", type=int)
# How old are you: <validates integer input>

color = click.prompt("Favorite color", default="blue")
# Favorite color [blue]: <Enter for default>

password = click.prompt("Password", hide_input=True, confirmation_prompt=True)
```

### `click.confirm()`

Yes/no confirmation:

```python
if click.confirm("Do you want to continue?"):
    click.echo("Continuing...")

# With abort (raises SystemExit on "no")
click.confirm("Delete everything?", abort=True)
click.echo("Deleting...")  # Only reached if user says yes
```

### `click.progressbar()`

Built-in progress bar:

```python
import time

items = range(100)
with click.progressbar(items, label="Processing") as bar:
    for item in bar:
        time.sleep(0.02)  # simulate work
```

```
Processing  [####################################]  100%
```

### `click.clear()`

Clear the terminal screen:

```python
click.clear()
```

### `click.edit()` and `click.launch()`

```python
# Open text in the user's editor ($EDITOR)
text = click.edit("Initial content here")

# Open a URL or file with the system handler
click.launch("https://example.com")
click.launch("/path/to/file.pdf")
```

---

## Summary Reference Table

### Decorators

| Decorator | Description |
|-----------|-------------|
| `@click.command()` | Define a command |
| `@click.group()` | Define a command group (with subcommands) |
| `@click.option()` | Add a `--option` parameter |
| `@click.argument()` | Add a positional argument |
| `@click.pass_context` | Pass the `click.Context` as the first argument |
| `@click.pass_obj` | Pass `ctx.obj` as the first argument |
| `@click.version_option()` | Add a `--version` option |
| `@click.help_option()` | Customize the `--help` option |
| `@click.confirmation_option()` | Add a `--yes` confirmation option |
| `@click.password_option()` | Add a `--password` option with hidden input |

### Output Functions

| Function | Description |
|----------|-------------|
| `click.echo(msg, err=False, nl=True)` | Print a message (encoding-safe) |
| `click.secho(msg, fg=None, bg=None, bold=False, ...)` | Print with ANSI styling |
| `click.style(msg, fg=None, bg=None, bold=False, ...)` | Apply ANSI styling (returns string) |
| `click.prompt(text, default=None, type=None, ...)` | Prompt for interactive input |
| `click.confirm(text, default=False, abort=False)` | Prompt for yes/no confirmation |
| `click.progressbar(iterable, label=None, ...)` | Display a progress bar |
| `click.clear()` | Clear the terminal |
| `click.edit(text=None, editor=None)` | Open text in an editor |
| `click.launch(url)` | Open a URL or file with the system handler |
| `click.get_terminal_size()` | Get terminal dimensions |

### Types

| Type | Description |
|------|-------------|
| `click.STRING` | Any string (default) |
| `click.INT` | Integer |
| `click.FLOAT` | Float |
| `click.BOOL` | Boolean (`true/false`, `yes/no`, `1/0`) |
| `click.UUID` | UUID |
| `click.Choice(values)` | One of a set of allowed string values |
| `click.IntRange(min, max)` | Integer within a range |
| `click.FloatRange(min, max)` | Float within a range |
| `click.DateTime(formats)` | Date/time from string |
| `click.Path(exists, ...)` | Filesystem path with validation |
| `click.File(mode)` | Opened file handle |
| `click.Tuple(types)` | Multi-value with heterogeneous types |
