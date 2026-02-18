# Typer — Integration & Testing

> Part of the typer skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Testing -- typer.testing.CliRunner](#testing----typertestingclirunner)
  - [Basic Test Pattern](#basic-test-pattern)
  - [CliRunner Methods and Result Object](#clirunner-methods-and-result-object)
  - [Testing with Prompts](#testing-with-prompts)
  - [Testing with Environment Variables](#testing-with-environment-variables)
- [Autocompletion -- Shell Completion Support](#autocompletion----shell-completion-support)
  - [Installing Completion](#installing-completion)
  - [Showing Completion Script](#showing-completion-script)
  - [Custom Autocompletion](#custom-autocompletion)
  - [Disabling Completion](#disabling-completion)
- [Rich Integration](#rich-integration)
  - [Rich Help Formatting](#rich-help-formatting)
  - [Rich Help Panels](#rich-help-panels)
  - [Pretty Exceptions](#pretty-exceptions)
  - [Using Rich Directly with Typer](#using-rich-directly-with-typer)
- [Version Callback Pattern](#version-callback-pattern)

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
