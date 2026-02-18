# Click — Commands & Groups

> Part of the click skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [@click.group() -- Creating Command Groups](#clickgroup----creating-command-groups)
- [@group.command() -- Registering Commands](#groupcommand----registering-commands)
- [Nesting Groups](#nesting-groups-groupgroup)
- [invoke_without_command](#invoke_without_command)
- [Command Chaining](#command-chaining)
- [Result Callbacks (Pipeline Pattern)](#result-callbacks-pipeline-pattern)
- [Context](#context)
- [@click.pass_context -- Accessing the Context](#clickpass_context----accessing-the-context)
- [@click.pass_obj -- Passing Objects Between Commands](#clickpass_obj----passing-objects-between-commands)
- [context_settings](#context_settings)
- [Invoking Commands from Code](#invoking-commands-from-code)

---

## `@click.group()` -- Creating Command Groups

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

## `@group.command()` -- Registering Commands

Commands are registered to groups using the `@group.command()` decorator:

```python
@cli.command()
def status():
    """Show status."""
    ...

# Or register an existing command:
cli.add_command(status)
```

## Nesting Groups (`@group.group()`)

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

## `invoke_without_command`

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

## Command Chaining

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

## Result Callbacks (Pipeline Pattern)

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

## `@click.pass_context` -- Accessing the Context

```python
@click.command()
@click.option("--debug/--no-debug", default=False)
@click.pass_context
def cli(ctx, debug):
    ctx.ensure_object(dict)
    ctx.obj["DEBUG"] = debug
    click.echo(f"Debug mode: {debug}")
```

## `@click.pass_obj` -- Passing Objects Between Commands

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

## `context_settings`

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

## Invoking Commands from Code

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
