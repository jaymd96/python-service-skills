# Click — Examples & Gotchas

> Part of the click skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
- [Complete Examples](#complete-examples)

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
