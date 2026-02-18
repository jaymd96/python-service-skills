# Click — Testing & Utilities

> Part of the click skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [click.testing.CliRunner](#clicktestingclirunner)
- [CliRunner API](#clirunner-api)
- [runner.invoke() Parameters](#runnerinvoke-parameters)
- [Result Object](#result-object)
- [Testing Examples](#testing-examples)
- [Testing Reference Table](#testing-reference-table)

---

## `click.testing.CliRunner`

Click provides a test runner that invokes commands in isolation without spawning a subprocess. This makes CLI tests fast and self-contained.

```python
from click.testing import CliRunner

def test_hello():
    runner = CliRunner()
    result = runner.invoke(hello, ["--name", "Alice"])

    assert result.exit_code == 0
    assert "Hello, Alice!" in result.output
```

## CliRunner API

```python
runner = CliRunner(
    charset="utf-8",       # Character encoding for I/O
    env=None,              # Dict of environment variables
    echo_input=False,      # Whether to echo input to output
    mix_stderr=True,       # If True, stderr is mixed into output; if False, separate
)
```

## `runner.invoke()` Parameters

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

## Result Object

```python
result.exit_code      # int: 0 for success, non-zero for failure
result.output         # str: captured stdout (and stderr if mix_stderr=True)
result.exception      # Exception or None: the exception if one was raised
result.runner         # The CliRunner instance
```

## Testing Examples

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

## Testing Reference Table

| API | Description |
|-----|-------------|
| `CliRunner()` | Create a test runner |
| `runner.invoke(cli, args, input, env)` | Invoke a command in isolation |
| `runner.isolated_filesystem()` | Context manager for temp working directory |
| `result.exit_code` | Exit code (0 = success) |
| `result.output` | Captured stdout |
| `result.exception` | Captured exception or None |
