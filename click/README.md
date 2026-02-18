# Click -- Composable Command-Line Interface Framework for Python

## Overview

**Click** (Command Line Interface Creation Kit) is a Python package for creating command-line interfaces with as little code as necessary. Developed and maintained by the Pallets project (the same team behind Flask), Click uses a decorator-based API that makes it simple to compose complex CLI applications from small, reusable pieces.

Click is the foundation that **Typer** builds on top of -- Typer adds type-hint-driven parameter declarations but delegates all actual CLI parsing, help generation, and shell completion to Click under the hood.

**Key Characteristics:**

- **Version:** 8.1.8 (latest stable as of early 2026)
- **Python:** 3.7+
- **License:** BSD 3-Clause
- **Dependencies:** None beyond the standard library (pure Python)
- **Repository:** https://github.com/pallets/click
- **Documentation:** https://click.palletsprojects.com

**When to use Click:**

- Building any command-line tool, from a single-command script to a multi-level CLI with dozens of subcommands
- When you want a decorator-based API that reads naturally and composes well
- When you need robust help page generation, type validation, and shell completion
- When building a CLI framework or tool that others will extend (Click's plugin architecture is excellent)
- When you need cross-platform compatibility (Click handles Windows console encoding, color support, and terminal differences)

**Core Design Principles:**

- **Composability** -- commands and groups nest arbitrarily; decorators stack cleanly
- **Convention over configuration** -- sensible defaults for help text, parameter names, error messages
- **Lazy evaluation** -- subcommands are loaded only when invoked, enabling fast startup for large CLIs
- **Explicit over implicit** -- no global state; context is passed explicitly through the `Context` object

---

## Installation

```bash
pip install click
```

Click has **zero runtime dependencies**. It is pure Python.

```bash
# Pin to latest stable
pip install click>=8.1,<9
```

---

## Core API

### `@click.command()` -- Defining Commands

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

#### `@click.command()` Parameters

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

### `@click.option()` -- Adding Options

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

#### Full `@click.option()` Parameter Reference

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

#### Common Option Patterns

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

### `@click.argument()` -- Adding Arguments

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

#### `@click.argument()` Parameter Reference

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

#### Variadic Arguments

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

#### Optional Arguments

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

#### File Arguments

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

### Parameter Types

Click has a rich type system for validating and converting parameter values.

#### Built-in Types

| Type | Description | Example |
|------|-------------|---------|
| `click.STRING` | Default. Any string. | `--name foo` |
| `click.INT` | Integer. | `--count 42` |
| `click.FLOAT` | Floating-point number. | `--rate 3.14` |
| `click.BOOL` | Boolean. Accepts `true/false`, `yes/no`, `1/0`. | `--flag true` |
| `click.UUID` | UUID string, converted to `uuid.UUID`. | `--id 12345678-...` |
| `click.UNPROCESSED` | Raw string, no processing. | -- |

#### Parameterized Types

| Type | Description | Example |
|------|-------------|---------|
| `click.Choice(choices, case_sensitive=True)` | Must be one of the listed values. | `Choice(["a", "b", "c"])` |
| `click.IntRange(min, max, min_open, max_open, clamp)` | Integer within a range. | `IntRange(0, 100)` |
| `click.FloatRange(min, max, min_open, max_open, clamp)` | Float within a range. | `FloatRange(0.0, 1.0)` |
| `click.DateTime(formats)` | Parses date/time strings. | `DateTime(["%Y-%m-%d"])` |
| `click.Path(exists, file_okay, dir_okay, readable, writable, resolve_path, path_type)` | Filesystem path with validation. | `Path(exists=True)` |
| `click.File(mode, encoding, errors, lazy, atomic)` | Opens a file handle. Supports `-` for stdin/stdout. | `File("r")` |
| `click.Tuple(types)` | Multiple values of specified types. | `Tuple([int, str])` |

#### `click.Path` in Detail

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

#### `click.DateTime`

```python
@click.option("--start", type=click.DateTime(formats=["%Y-%m-%d", "%Y-%m-%dT%H:%M:%S"]))
def report(start):
    click.echo(f"Report starting from {start}")
```

```
$ python cli.py --start 2025-01-15
Report starting from 2025-01-15 00:00:00
```

#### Custom Types

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

### Callbacks and Value Processing

#### Option Callbacks

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

#### Eager Options

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

### Output Functions

#### `click.echo()`

The primary output function. Preferred over `print()` because it handles encoding issues, supports color, and works correctly across platforms including broken pipe scenarios.

```python
click.echo("Hello, World!")                      # Print to stdout
click.echo("Error!", err=True)                   # Print to stderr
click.echo("No newline", nl=False)               # No trailing newline
click.echo(click.style("Bold!", bold=True))      # Styled output
click.echo(click.style("Error!", fg="red"), err=True)  # Colored stderr
```

#### `click.style()` and `click.secho()`

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

#### `click.prompt()`

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

#### `click.confirm()`

Yes/no confirmation:

```python
if click.confirm("Do you want to continue?"):
    click.echo("Continuing...")

# With abort (raises SystemExit on "no")
click.confirm("Delete everything?", abort=True)
click.echo("Deleting...")  # Only reached if user says yes
```

#### `click.progressbar()`

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

#### `click.clear()`

Clear the terminal screen:

```python
click.clear()
```

#### `click.edit()` and `click.launch()`

```python
# Open text in the user's editor ($EDITOR)
text = click.edit("Initial content here")

# Open a URL or file with the system handler
click.launch("https://example.com")
click.launch("/path/to/file.pdf")
```

---

## Groups and Subcommands

### `@click.group()` -- Creating Command Groups

Groups are commands that contain subcommands. This is how you build multi-command CLIs like `git`, `docker`, or `pip`.

```python
@click.group()
def cli():
    """A multi-command CLI application."""
    pass

@cli.command()
def init():
    """Initialize a new project."""
    click.echo("Initialized!")

@cli.command()
@click.argument("name")
def greet(name):
    """Greet someone by NAME."""
    click.echo(f"Hello, {name}!")

if __name__ == "__main__":
    cli()
```

```
$ python cli.py --help
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

  A multi-command CLI application.

Options:
  --help  Show this message and exit.

Commands:
  greet  Greet someone by NAME.
  init   Initialize a new project.

$ python cli.py init
Initialized!

$ python cli.py greet Alice
Hello, Alice!
```

### `@group.command()` -- Registering Commands

Commands are registered to groups using the `@group.command()` decorator:

```python
@cli.command()
def status():
    """Show status."""
    ...

# Or register an existing command:
cli.add_command(status)
```

### Nesting Groups (`@group.group()`)

Groups can contain other groups for deep nesting:

```python
@click.group()
def cli():
    """Top-level CLI."""
    pass

@cli.group()
def db():
    """Database commands."""
    pass

@db.command()
def migrate():
    """Run database migrations."""
    click.echo("Running migrations...")

@db.command()
def seed():
    """Seed the database."""
    click.echo("Seeding...")

@cli.group()
def user():
    """User management commands."""
    pass

@user.command()
@click.argument("username")
def create(username):
    """Create a new user."""
    click.echo(f"Created user: {username}")
```

```
$ python cli.py db migrate
Running migrations...

$ python cli.py user create alice
Created user: alice

$ python cli.py db --help
Usage: cli.py db [OPTIONS] COMMAND [ARGS]...

  Database commands.

Options:
  --help  Show this message and exit.

Commands:
  migrate  Run database migrations.
  seed     Seed the database.
```

### `invoke_without_command`

By default, a group requires a subcommand. Set `invoke_without_command=True` to allow the group itself to execute:

```python
@click.group(invoke_without_command=True)
@click.pass_context
def cli(ctx):
    """CLI with optional subcommand."""
    if ctx.invoked_subcommand is None:
        click.echo("No subcommand given. Showing status...")
    else:
        click.echo(f"About to invoke: {ctx.invoked_subcommand}")

@cli.command()
def deploy():
    """Deploy the application."""
    click.echo("Deploying!")
```

```
$ python cli.py
No subcommand given. Showing status...

$ python cli.py deploy
About to invoke: deploy
Deploying!
```

### Command Chaining

Enable `chain=True` to allow invoking multiple subcommands in sequence:

```python
@click.group(chain=True)
def pipeline():
    """Run processing steps in sequence."""
    pass

@pipeline.command()
def clean():
    """Clean the data."""
    click.echo("Cleaning...")

@pipeline.command()
def transform():
    """Transform the data."""
    click.echo("Transforming...")

@pipeline.command()
def load():
    """Load the data."""
    click.echo("Loading...")
```

```
$ python cli.py clean transform load
Cleaning...
Transforming...
Loading...
```

### Result Callbacks (Pipeline Pattern)

Combine chaining with result callbacks to pass data through a pipeline of commands:

```python
@click.group(chain=True, invoke_without_command=True)
def pipeline():
    """Data processing pipeline."""
    pass

@pipeline.result_callback()
def process_pipeline(processors):
    """Process data through all chained commands."""
    # Each chained command returns a function; apply them in sequence
    data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    for processor in processors:
        data = processor(data)
    click.echo(f"Result: {data}")

@pipeline.command("filter-even")
def filter_even():
    """Keep only even numbers."""
    return lambda data: [x for x in data if x % 2 == 0]

@pipeline.command("double")
def double():
    """Double each value."""
    return lambda data: [x * 2 for x in data]

@pipeline.command("sum")
def sum_all():
    """Sum all values."""
    return lambda data: sum(data)
```

```
$ python cli.py filter-even double sum
Result: 60
```

---

## Context

### `click.Context` -- The Heart of Click

Every Click command execution creates a `Context` object that carries state through the invocation. It holds the parsed parameters, the parent context (for subcommands), and an arbitrary object store for passing data between commands.

#### Key `Context` Attributes

| Attribute | Description |
|-----------|-------------|
| `ctx.params` | Dict of parsed parameter values |
| `ctx.parent` | Parent context (for subcommands) |
| `ctx.obj` | Arbitrary user object for passing data between commands |
| `ctx.info_name` | The command name as shown in help |
| `ctx.invoked_subcommand` | Name of the subcommand about to be invoked |
| `ctx.command` | The `Command` instance being executed |
| `ctx.color` | Whether output should use ANSI colors |
| `ctx.resilient_parsing` | If True, the parser is in "completion mode" (do not execute callbacks) |

### `@click.pass_context` -- Accessing the Context

```python
@click.command()
@click.option("--debug/--no-debug", default=False)
@click.pass_context
def cli(ctx, debug):
    ctx.ensure_object(dict)
    ctx.obj["DEBUG"] = debug
    click.echo(f"Debug mode: {debug}")
```

### `@click.pass_obj` -- Passing Objects Between Commands

The most common pattern for sharing state across a group and its subcommands:

```python
import click

class Config:
    def __init__(self):
        self.debug = False
        self.database_url = ""

@click.group()
@click.option("--debug/--no-debug", default=False)
@click.option("--db", envvar="DATABASE_URL", default="sqlite:///app.db")
@click.pass_context
def cli(ctx, debug, db):
    """CLI with shared configuration."""
    ctx.ensure_object(Config)
    ctx.obj.debug = debug
    ctx.obj.database_url = db

@cli.command()
@click.pass_obj
def status(config: Config):
    """Show application status."""
    click.echo(f"Debug: {config.debug}")
    click.echo(f"Database: {config.database_url}")

@cli.command()
@click.argument("query")
@click.pass_obj
def query(config: Config, query):
    """Run a database QUERY."""
    if config.debug:
        click.echo(f"[DEBUG] Connecting to {config.database_url}")
    click.echo(f"Running: {query}")
```

```
$ python cli.py --debug status
Debug: True
Database: sqlite:///app.db

$ python cli.py --db postgres://localhost/mydb query "SELECT 1"
Running: SELECT 1
```

### `context_settings`

Pass settings to the `Context` constructor via `context_settings`:

```python
CONTEXT_SETTINGS = dict(
    help_option_names=["-h", "--help"],  # Add -h as help alias
    max_content_width=120,               # Wider help output
    show_default=True,                   # Show defaults for all options
    auto_envvar_prefix="MYAPP",          # Auto-read MYAPP_* env vars
)

@click.command(context_settings=CONTEXT_SETTINGS)
@click.option("--port", default=8080, type=int)
def serve(port):
    """Start the server."""
    click.echo(f"Serving on port {port}")
```

With `auto_envvar_prefix="MYAPP"`, the `--port` option automatically reads from the `MYAPP_PORT` environment variable.

### Invoking Commands from Code

You can invoke other commands programmatically from within a command:

```python
@click.group()
@click.pass_context
def cli(ctx):
    pass

@cli.command()
@click.pass_context
def deploy(ctx):
    """Deploy the app."""
    click.echo("Building first...")
    ctx.invoke(build)  # Invoke another command
    click.echo("Deploying...")

@cli.command()
def build():
    """Build the app."""
    click.echo("Building!")
```

You can pass arguments to `ctx.invoke`:

```python
ctx.invoke(build, target="production", optimize=True)
```

---

## Testing

### `click.testing.CliRunner`

Click provides a test runner that invokes commands in isolation without spawning a subprocess. This makes CLI tests fast and self-contained.

```python
from click.testing import CliRunner

def test_hello():
    runner = CliRunner()
    result = runner.invoke(hello, ["--name", "Alice"])

    assert result.exit_code == 0
    assert "Hello, Alice!" in result.output
```

#### `CliRunner` API

```python
runner = CliRunner(
    charset="utf-8",       # Character encoding for I/O
    env=None,              # Dict of environment variables
    echo_input=False,      # Whether to echo input to output
    mix_stderr=True,       # If True, stderr is mixed into output; if False, separate
)
```

#### `runner.invoke()` Parameters

```python
result = runner.invoke(
    cli,                   # The Click command or group to invoke
    args=["--flag", "val"],# List of string arguments (simulates sys.argv)
    input="yes\n",         # Simulated stdin input (for prompts)
    env={"KEY": "value"},  # Environment variable overrides
    catch_exceptions=True, # Whether to catch exceptions (set False for debugging)
    color=False,           # Whether output includes ANSI color codes
    standalone_mode=None,  # Override standalone_mode for the invocation
)
```

#### `Result` Object

```python
result.exit_code      # int: 0 for success, non-zero for failure
result.output         # str: captured stdout (and stderr if mix_stderr=True)
result.exception      # Exception or None: the exception if one was raised
result.runner         # The CliRunner instance
```

#### Testing Examples

**Testing basic command:**

```python
from click.testing import CliRunner

@click.command()
@click.option("--name", default="World")
def greet(name):
    click.echo(f"Hello, {name}!")

def test_greet_default():
    runner = CliRunner()
    result = runner.invoke(greet)
    assert result.exit_code == 0
    assert result.output == "Hello, World!\n"

def test_greet_with_name():
    runner = CliRunner()
    result = runner.invoke(greet, ["--name", "Alice"])
    assert result.exit_code == 0
    assert "Alice" in result.output
```

**Testing prompts:**

```python
@click.command()
@click.option("--name", prompt="Your name")
def greet(name):
    click.echo(f"Hello, {name}!")

def test_greet_prompt():
    runner = CliRunner()
    result = runner.invoke(greet, input="Alice\n")
    assert result.exit_code == 0
    assert "Hello, Alice!" in result.output
```

**Testing groups and subcommands:**

```python
@click.group()
def cli():
    pass

@cli.command()
@click.argument("name")
def create(name):
    click.echo(f"Created: {name}")

def test_create_subcommand():
    runner = CliRunner()
    result = runner.invoke(cli, ["create", "my-project"])
    assert result.exit_code == 0
    assert "Created: my-project" in result.output
```

**Testing with environment variables:**

```python
@click.command()
@click.option("--api-key", envvar="API_KEY", required=True)
def fetch(api_key):
    click.echo(f"Using key: {api_key[:4]}...")

def test_fetch_with_env():
    runner = CliRunner()
    result = runner.invoke(fetch, env={"API_KEY": "secret123"})
    assert result.exit_code == 0
    assert "secr..." in result.output
```

**Testing file I/O with isolated filesystem:**

```python
@click.command()
@click.argument("output", type=click.Path())
def export(output):
    with open(output, "w") as f:
        f.write("data")
    click.echo(f"Wrote to {output}")

def test_export():
    runner = CliRunner()
    with runner.isolated_filesystem():
        result = runner.invoke(export, ["out.txt"])
        assert result.exit_code == 0
        with open("out.txt") as f:
            assert f.read() == "data"
```

**Debugging test failures:**

```python
def test_debug():
    runner = CliRunner()
    result = runner.invoke(cli, ["bad-arg"], catch_exceptions=False)
    # If an exception occurs, it will be raised directly
    # instead of being caught and stored in result.exception

    # For detailed debugging:
    if result.exception:
        import traceback
        traceback.print_exception(type(result.exception), result.exception, result.exception.__traceback__)
```

---

## Gotchas and Common Mistakes

### 1. Decorator Order Matters

Click decorators must be applied in a specific order: `@click.command()` must be the **outermost** (topmost) decorator, and options/arguments are applied bottom-to-top (because Python applies decorators from the inside out).

```python
# CORRECT: @click.command() on top, arguments/options below
@click.command()
@click.option("--count", default=1)
@click.argument("name")
def greet(name, count):
    ...

# WRONG: @click.command() is not outermost
@click.option("--count", default=1)
@click.command()
@click.argument("name")
def greet(name, count):
    ...
# This will produce confusing errors or silently misbehave
```

The function parameters correspond to the decorated parameters by name, not by decorator order.

### 2. Boolean Flags -- The `--flag/--no-flag` Pattern

The `--flag/--no-flag` syntax creates a boolean pair:

```python
@click.option("--verbose/--no-verbose", default=False)
```

If you use `is_flag=True` instead, there is no way to explicitly pass `False`:

```python
@click.option("--verbose", is_flag=True)
# Only --verbose is available, not --no-verbose
```

Common mistake -- forgetting that `--flag/--no-flag` creates a single parameter with the name derived from the flag portion:

```python
@click.option("--verbose/--no-verbose")
def cli(verbose):  # Parameter name is "verbose", not "no_verbose"
    ...
```

When using a secondary name, the parameter name is derived from the longest flag:

```python
@click.option("-v/--no-verbose")
def cli(v):  # Parameter name is "v" (longest is -v vs --no-verbose, but primary name wins)
    ...
```

### 3. `standalone_mode` and SystemExit

By default, Click commands run in `standalone_mode=True`, which means:
- The command calls `sys.exit(0)` on success
- Exceptions are caught, printed, and cause `sys.exit(1)`
- `--help` causes `sys.exit(0)`

This is fine for CLI entry points but causes problems when calling commands programmatically:

```python
# This raises SystemExit!
result = cli(standalone_mode=False)  # Fix: pass standalone_mode=False

# Or use CliRunner for testing (it handles standalone_mode internally)
runner = CliRunner()
result = runner.invoke(cli, ["--help"])
# result.exit_code == 0, no SystemExit raised
```

When invoking commands from within other Click commands, use `ctx.invoke()` which handles this correctly.

### 4. Unicode and Encoding

Click handles encoding carefully, which is why `click.echo()` should be preferred over `print()`:

```python
# BAD: can fail on Windows with non-UTF-8 console
print("Hello \u2603")  # Snowman character

# GOOD: Click handles encoding gracefully
click.echo("Hello \u2603")
```

`click.echo()` uses the `errors="replace"` strategy when the output stream does not support the character, preventing `UnicodeEncodeError` crashes.

### 5. Windows Compatibility

- **Color output:** Click auto-detects color support. On Windows, it uses `colorama` if available (installed as a dependency on Windows). Without colorama, ANSI codes are stripped.
- **File paths:** Use `click.Path()` and `click.File()` instead of raw strings for cross-platform path handling.
- **Binary mode:** When reading/writing binary data, use `click.get_binary_stream("stdin")` instead of `sys.stdin.buffer`:

```python
# Cross-platform binary stdin
stdin = click.get_binary_stream("stdin")
stdout = click.get_binary_stream("stdout")
```

### 6. Arguments Do Not Appear in `--help` by Default

Unlike options, arguments are not documented in the help text by default. You must describe them in the docstring:

```python
@click.command()
@click.argument("filename")
def process(filename):
    """Process the given FILENAME.

    FILENAME is the path to the input file to process.
    """
    ...
```

### 7. Multiple Values and Tuple Unpacking

When using `nargs` or `multiple`, be mindful of the resulting types:

```python
@click.command()
@click.option("--point", nargs=2, type=float)      # Receives a tuple: (1.0, 2.0)
@click.option("--include", multiple=True)            # Receives a tuple: ("a", "b")
@click.argument("files", nargs=-1)                   # Receives a tuple: ("f1", "f2")
def cmd(point, include, files):
    # point is a tuple of 2 floats, or None if not provided
    # include is a tuple of strings (empty tuple if not provided)
    # files is a tuple of strings (empty tuple if none given)
    ...
```

### 8. Default Values and Type Inference

Click infers the parameter type from the default value. This can be surprising:

```python
@click.option("--count", default=1)          # type=INT (inferred from int default)
@click.option("--name", default="World")     # type=STRING (inferred)
@click.option("--rate", default=0.5)         # type=FLOAT (inferred)
@click.option("--verbose", default=False)    # type=BOOL (inferred)
@click.option("--limit", default=None)       # type=STRING (None doesn't infer)
```

When the default is `None`, Click defaults to `STRING`. Explicitly set the type if you want something else:

```python
@click.option("--limit", default=None, type=int)  # Explicit type
```

### 9. Resilient Parsing in Callbacks

When implementing eager callbacks (like `--version`), always check `ctx.resilient_parsing`. This is `True` during shell completion, and you should not execute side effects:

```python
def version_callback(ctx, param, value):
    if not value or ctx.resilient_parsing:
        return  # Do nothing during shell completion
    click.echo("1.0.0")
    ctx.exit()
```

Without this check, shell completion triggers the version output.

### 10. Entry Points and `setuptools`

For installable CLI tools, define entry points in `pyproject.toml`:

```toml
[project.scripts]
my-tool = "mypackage.cli:cli"
```

Or in `setup.py`:

```python
setup(
    entry_points={
        "console_scripts": [
            "my-tool=mypackage.cli:cli",
        ],
    },
)
```

The entry point should reference the Click group or command object, not the module.

---

## Complete Examples

### Example 1: Basic CLI with Options and Arguments

A file processing tool with common CLI patterns:

```python
import click
from pathlib import Path

@click.command()
@click.argument("input_file", type=click.Path(exists=True, path_type=Path))
@click.option("-o", "--output", type=click.Path(path_type=Path), default=None,
              help="Output file path. Defaults to stdout.")
@click.option("-f", "--format", "fmt", type=click.Choice(["json", "csv", "table"]),
              default="table", show_default=True, help="Output format.")
@click.option("-n", "--limit", type=click.IntRange(min=1), default=None,
              help="Limit number of rows.")
@click.option("-v", "--verbose", count=True, help="Increase verbosity (-v, -vv, -vvv).")
@click.version_option(version="1.0.0")
def convert(input_file, output, fmt, limit, verbose):
    """Convert INPUT_FILE to a different format.

    INPUT_FILE is the path to the data file to convert. Supports CSV and JSON
    input files.
    """
    if verbose >= 1:
        click.secho(f"Reading {input_file}...", fg="blue")

    # Simulate reading data
    data = input_file.read_text()
    lines = data.strip().split("\n")

    if limit:
        lines = lines[:limit]

    if verbose >= 2:
        click.secho(f"Processing {len(lines)} lines in {fmt} format...", fg="blue")

    result = "\n".join(lines)  # Simplified transform

    if output:
        output.write_text(result)
        if verbose >= 1:
            click.secho(f"Wrote to {output}", fg="green")
    else:
        click.echo(result)

if __name__ == "__main__":
    convert()
```

### Example 2: Multi-Command Application

A project management CLI with nested groups and shared configuration:

```python
import click
import json
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AppContext:
    """Shared application context passed between commands."""
    config_path: Path = Path(".project.json")
    verbose: bool = False
    config: dict = field(default_factory=dict)

    def load_config(self):
        if self.config_path.exists():
            self.config = json.loads(self.config_path.read_text())
        return self.config

    def save_config(self):
        self.config_path.write_text(json.dumps(self.config, indent=2))


pass_app = click.make_pass_decorator(AppContext, ensure=True)


@click.group()
@click.option("--config", "config_path", type=click.Path(path_type=Path),
              default=".project.json", show_default=True,
              help="Path to config file.")
@click.option("-v", "--verbose", is_flag=True, help="Enable verbose output.")
@pass_app
def cli(app: AppContext, config_path: Path, verbose: bool):
    """Project management CLI.

    Manage projects, tasks, and deployments from the command line.
    """
    app.config_path = config_path
    app.verbose = verbose
    app.load_config()


# --- Project commands ---

@cli.command()
@click.argument("name")
@click.option("--description", "-d", default="", help="Project description.")
@pass_app
def init(app: AppContext, name: str, description: str):
    """Initialize a new project with NAME."""
    if app.config_path.exists():
        click.confirm(f"{app.config_path} already exists. Overwrite?", abort=True)

    app.config = {
        "name": name,
        "description": description,
        "tasks": [],
    }
    app.save_config()
    click.secho(f"Initialized project '{name}'", fg="green")


@cli.command()
@pass_app
def status(app: AppContext):
    """Show project status."""
    if not app.config:
        click.secho("No project found. Run 'init' first.", fg="red", err=True)
        raise SystemExit(1)

    click.secho(f"Project: {app.config['name']}", bold=True)
    if app.config.get("description"):
        click.echo(f"  {app.config['description']}")

    tasks = app.config.get("tasks", [])
    done = sum(1 for t in tasks if t.get("done"))
    click.echo(f"  Tasks: {done}/{len(tasks)} complete")


# --- Task subgroup ---

@cli.group()
def task():
    """Manage project tasks."""
    pass


@task.command("add")
@click.argument("title")
@click.option("--priority", type=click.Choice(["low", "medium", "high"]),
              default="medium", show_default=True)
@pass_app
def task_add(app: AppContext, title: str, priority: str):
    """Add a new task with TITLE."""
    new_task = {
        "id": len(app.config.get("tasks", [])) + 1,
        "title": title,
        "priority": priority,
        "done": False,
    }
    app.config.setdefault("tasks", []).append(new_task)
    app.save_config()
    click.secho(f"Added task #{new_task['id']}: {title}", fg="green")


@task.command("list")
@click.option("--all", "show_all", is_flag=True, help="Show completed tasks too.")
@pass_app
def task_list(app: AppContext, show_all: bool):
    """List all tasks."""
    tasks = app.config.get("tasks", [])
    if not tasks:
        click.echo("No tasks yet.")
        return

    for t in tasks:
        if not show_all and t.get("done"):
            continue
        status_mark = click.style("done", fg="green") if t["done"] else click.style("todo", fg="yellow")
        click.echo(f"  [{status_mark}] #{t['id']} {t['title']} ({t['priority']})")


@task.command("done")
@click.argument("task_id", type=int)
@pass_app
def task_done(app: AppContext, task_id: int):
    """Mark task TASK_ID as done."""
    tasks = app.config.get("tasks", [])
    for t in tasks:
        if t["id"] == task_id:
            t["done"] = True
            app.save_config()
            click.secho(f"Task #{task_id} marked as done.", fg="green")
            return
    click.secho(f"Task #{task_id} not found.", fg="red", err=True)
    raise SystemExit(1)


if __name__ == "__main__":
    cli()
```

Usage:

```
$ python cli.py init my-project -d "A sample project"
Initialized project 'my-project'

$ python cli.py task add "Write documentation" --priority high
Added task #1: Write documentation

$ python cli.py task add "Write tests"
Added task #2: Write tests

$ python cli.py task list
  [todo] #1 Write documentation (high)
  [todo] #2 Write tests (medium)

$ python cli.py task done 1
Task #1 marked as done.

$ python cli.py status
Project: my-project
  A sample project
  Tasks: 1/2 complete
```

### Example 3: Testing the Multi-Command Application

```python
import json
import pytest
from click.testing import CliRunner
# Assuming the CLI from Example 2 is importable
from myapp.cli import cli


@pytest.fixture
def runner():
    return CliRunner()


@pytest.fixture
def project_dir(runner):
    """Set up an isolated filesystem with an initialized project."""
    with runner.isolated_filesystem() as td:
        result = runner.invoke(cli, ["init", "test-project", "-d", "Test"])
        assert result.exit_code == 0
        yield td


class TestInit:
    def test_creates_config_file(self, runner):
        with runner.isolated_filesystem():
            result = runner.invoke(cli, ["init", "my-project"])
            assert result.exit_code == 0
            assert "Initialized" in result.output

            config = json.loads(open(".project.json").read())
            assert config["name"] == "my-project"

    def test_confirms_overwrite(self, runner, project_dir):
        # Decline overwrite
        result = runner.invoke(cli, ["init", "new-name"], input="n\n")
        assert result.exit_code != 0  # Aborted

        # Accept overwrite
        result = runner.invoke(cli, ["init", "new-name"], input="y\n")
        assert result.exit_code == 0


class TestStatus:
    def test_shows_project_info(self, runner, project_dir):
        result = runner.invoke(cli, ["status"])
        assert result.exit_code == 0
        assert "test-project" in result.output
        assert "0/0 complete" in result.output

    def test_no_project_error(self, runner):
        with runner.isolated_filesystem():
            result = runner.invoke(cli, ["status"])
            assert result.exit_code != 0


class TestTasks:
    def test_add_and_list(self, runner, project_dir):
        runner.invoke(cli, ["task", "add", "Task 1"])
        runner.invoke(cli, ["task", "add", "Task 2", "--priority", "high"])

        result = runner.invoke(cli, ["task", "list"])
        assert result.exit_code == 0
        assert "Task 1" in result.output
        assert "Task 2" in result.output
        assert "high" in result.output

    def test_mark_done(self, runner, project_dir):
        runner.invoke(cli, ["task", "add", "Task 1"])
        result = runner.invoke(cli, ["task", "done", "1"])
        assert result.exit_code == 0
        assert "marked as done" in result.output

    def test_done_hides_completed(self, runner, project_dir):
        runner.invoke(cli, ["task", "add", "Task 1"])
        runner.invoke(cli, ["task", "add", "Task 2"])
        runner.invoke(cli, ["task", "done", "1"])

        # Without --all, completed tasks are hidden
        result = runner.invoke(cli, ["task", "list"])
        assert "Task 1" not in result.output
        assert "Task 2" in result.output

        # With --all, all tasks are shown
        result = runner.invoke(cli, ["task", "list", "--all"])
        assert "Task 1" in result.output
        assert "Task 2" in result.output

    def test_done_nonexistent_task(self, runner, project_dir):
        result = runner.invoke(cli, ["task", "done", "999"])
        assert result.exit_code != 0
        assert "not found" in result.output


class TestVerbose:
    def test_verbose_flag_passes_to_context(self, runner, project_dir):
        result = runner.invoke(cli, ["-v", "status"])
        assert result.exit_code == 0


class TestHelp:
    def test_top_level_help(self, runner):
        result = runner.invoke(cli, ["--help"])
        assert result.exit_code == 0
        assert "Project management CLI" in result.output

    def test_subcommand_help(self, runner):
        result = runner.invoke(cli, ["task", "--help"])
        assert result.exit_code == 0
        assert "Manage project tasks" in result.output

    def test_nested_command_help(self, runner):
        result = runner.invoke(cli, ["task", "add", "--help"])
        assert result.exit_code == 0
        assert "TITLE" in result.output
        assert "--priority" in result.output
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

### Testing

| API | Description |
|-----|-------------|
| `CliRunner()` | Create a test runner |
| `runner.invoke(cli, args, input, env)` | Invoke a command in isolation |
| `runner.isolated_filesystem()` | Context manager for temp working directory |
| `result.exit_code` | Exit code (0 = success) |
| `result.output` | Captured stdout |
| `result.exception` | Captured exception or None |
