# Typer — Subcommands & Groups

> Part of the typer skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [app.add\_typer(sub\_app, name=...) -- Nesting Command Groups](#appadd_typersub_app-name----nesting-command-groups)
- [Shared Options via Callback](#shared-options-via-callback)
- [Deep Nesting](#deep-nesting)
- [Click Interop -- Accessing the Underlying Click Layer](#click-interop----accessing-the-underlying-click-layer)
  - [Getting the Click Command Object](#getting-the-click-command-object)
  - [Using Click Decorators Alongside Typer](#using-click-decorators-alongside-typer)
  - [Using Click Types Not Wrapped by Typer](#using-click-types-not-wrapped-by-typer)
  - [Registering Click Commands in a Typer App](#registering-click-commands-in-a-typer-app)
  - [Entry Points (setup.py / pyproject.toml)](#entry-points-setuppy--pyprojecttoml)

## `app.add_typer(sub_app, name=...)` -- Nesting Command Groups

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

## Shared Options via Callback

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

## Deep Nesting

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
