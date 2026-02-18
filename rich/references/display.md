# Rich — Display & Integration

> Part of the rich skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Rich Inspect](#rich-inspect)
- [RichHandler -- As a Logging Handler](#richhandler----as-a-logging-handler)
- [With Click / Typer](#with-click--typer)
- [rich.traceback.install() -- Default Traceback Handler](#richtraceback-install----default-traceback-handler)
- [Traceback -- Beautiful Tracebacks](#traceback----beautiful-tracebacks)
- [Capturing Output](#capturing-output)
- [Quick Reference](#quick-reference)
- [Further Reading](#further-reading)

---

## Rich Inspect

`rich.inspect()` is a powerful introspection tool that displays detailed information about any Python object, including its methods, properties, documentation, and source.

```python
import rich

# Inspect a list
rich.inspect([1, 2, 3])

# Show methods
rich.inspect([1, 2, 3], methods=True)

# Show all (including private/dunder methods)
rich.inspect([1, 2, 3], all=True)

# Show help/docstrings
rich.inspect(str, help=True)

# Show the source code of a function
rich.inspect(my_function, methods=True, docs=True)
```

**Parameters for `rich.inspect()`:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `obj` | `Any` | -- | Object to inspect |
| `console` | `Console \| None` | `None` | Console to use |
| `title` | `str \| None` | `None` | Override the title |
| `help` | `bool` | `False` | Show full docstrings |
| `methods` | `bool` | `False` | Show methods |
| `docs` | `bool` | `True` | Show brief documentation |
| `private` | `bool` | `False` | Show private attributes (single underscore) |
| `dunder` | `bool` | `False` | Show dunder methods |
| `all` | `bool` | `False` | Show all attributes (equivalent to `private=True, dunder=True`) |
| `sort` | `bool` | `True` | Sort attributes alphabetically |
| `value` | `bool` | `True` | Show attribute values |

---

## `RichHandler` -- As a Logging Handler

Rich provides a logging handler that produces beautiful, colorful log output.

```python
import logging
from rich.logging import RichHandler

# Basic setup
logging.basicConfig(
    level=logging.DEBUG,
    format="%(message)s",
    datefmt="[%X]",
    handlers=[RichHandler()],
)

log = logging.getLogger("rich")
log.info("Server started on port 8000")
log.warning("Cache miss for key: user_123")
log.error("Database connection failed")
log.debug("Query executed in 0.003s")
```

**`RichHandler` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `level` | `int` | `NOTSET` | Logging level |
| `console` | `Console \| None` | `None` | Console to write to |
| `show_time` | `bool` | `True` | Show timestamps |
| `show_level` | `bool` | `True` | Show log level |
| `show_path` | `bool` | `True` | Show file path and line number |
| `enable_link_path` | `bool` | `True` | Make paths clickable links |
| `markup` | `bool` | `False` | Enable Rich markup in log messages |
| `rich_tracebacks` | `bool` | `False` | Render tracebacks with Rich |
| `tracebacks_show_locals` | `bool` | `False` | Show local variables in tracebacks |
| `tracebacks_extra_lines` | `int` | `3` | Extra context lines in tracebacks |
| `tracebacks_theme` | `str \| None` | `None` | Pygments theme for tracebacks |
| `tracebacks_word_wrap` | `bool` | `True` | Word wrap in tracebacks |
| `tracebacks_suppress` | `list[str]` | `[]` | Module paths to suppress in tracebacks |
| `log_time_format` | `str \| Callable` | `"[%X]"` | Time format (strftime or callable) |
| `highlighter` | `Highlighter \| None` | `ReprHighlighter()` | Highlighter for log messages |
| `keywords` | `list[str] \| None` | `None` | Additional keywords to highlight |

**Enable Rich markup in log messages:**

```python
handler = RichHandler(markup=True, rich_tracebacks=True)
logging.basicConfig(level="DEBUG", handlers=[handler])

log = logging.getLogger("app")
log.info("[bold green]Server started[/bold green] on port [cyan]8000[/cyan]")
```

---

## With Click / Typer

Rich works naturally with CLI frameworks. Use `rich.print` as a drop-in or pass a Console to your CLI.

**With Click:**

```python
import click
from rich.console import Console

console = Console()

@click.command()
@click.argument("name")
def hello(name: str):
    console.print(f"Hello, [bold magenta]{name}[/bold magenta]!")
    console.print(Panel(f"Welcome, {name}"))

if __name__ == "__main__":
    hello()
```

**With Typer (built-in Rich support):**

```python
import typer
from rich import print as rprint
from rich.console import Console
from rich.table import Table

app = typer.Typer()
console = Console()

@app.command()
def users():
    table = Table(title="Users")
    table.add_column("Name")
    table.add_column("Email")
    table.add_row("Alice", "alice@example.com")
    table.add_row("Bob", "bob@example.com")
    console.print(table)

if __name__ == "__main__":
    app()
```

---

## `rich.traceback.install()` -- Default Traceback Handler

Install Rich as the default traceback handler for all uncaught exceptions:

```python
from rich.traceback import install
install(show_locals=True, suppress=["click"])

# Now any uncaught exception will be rendered with Rich
```

This sets `sys.excepthook` to use Rich's traceback renderer. It is recommended to call this at application startup. The `suppress` parameter lets you hide frames from third-party libraries to reduce noise.

---

## `Traceback` -- Beautiful Tracebacks

Rich can render Python tracebacks with syntax highlighting, local variables, and clean formatting.

```python
from rich.traceback import install

# Install globally -- all uncaught exceptions get rich tracebacks
install(show_locals=True)

# Or render a specific traceback
from rich.console import Console
from rich.traceback import Traceback

console = Console()
try:
    1 / 0
except Exception:
    console.print_exception(show_locals=True)

# Or create a Traceback renderable manually
tb = Traceback()
console.print(tb)
```

**`install()` / `Traceback` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `show_locals` | `bool` | `False` | Show local variables in each frame |
| `width` | `int \| None` | `100` | Width of the traceback |
| `extra_lines` | `int` | `3` | Extra lines of code context around the error |
| `theme` | `str \| None` | `None` | Pygments theme for code |
| `word_wrap` | `bool` | `False` | Wrap long lines |
| `max_frames` | `int` | `100` | Max frames to show |
| `suppress` | `list[str]` | `[]` | Module paths to suppress from output |

**Suppressing library frames:**

```python
install(show_locals=True, suppress=["click", "fastapi", "uvicorn"])
```

---

## Capturing Output

```python
from rich.console import Console
import io

# WRONG: captures ANSI codes
buffer = io.StringIO()
console = Console(file=buffer)
console.print("[bold]Hello[/bold]")
raw = buffer.getvalue()  # Contains ANSI escape sequences

# RIGHT: use Console.capture() for plain text
console = Console()
with console.capture() as capture:
    console.print("[bold]Hello[/bold]")
plain_text = capture.get()  # "Hello\n" (no ANSI codes)

# RIGHT: use record=True for HTML/SVG/text export
console = Console(record=True)
console.print("[bold red]Error![/bold red]")
html = console.export_html()
text = console.export_text()
svg = console.export_svg(title="Output")
```

---

## Quick Reference

| What you want | How to do it |
|---------------|-------------|
| Drop-in `print()` replacement | `from rich import print` |
| Styled output | `console.print("[bold red]text[/]")` |
| Pretty-print data | `console.print(data)` or `from rich.pretty import pprint` |
| Table | `Table()` + `add_column()` + `add_row()` |
| Progress bar | `from rich.progress import track; for x in track(iterable): ...` |
| Spinner | `with console.status("Working..."): ...` |
| Syntax highlighting | `Syntax(code, "python")` |
| Tree view | `Tree("root")` + `.add("child")` |
| Panel (boxed content) | `Panel("content", title="Title")` |
| Markdown | `Markdown("# Heading")` |
| Logging | `logging.basicConfig(handlers=[RichHandler()])` |
| Pretty tracebacks | `from rich.traceback import install; install()` |
| Object inspection | `rich.inspect(obj, methods=True)` |
| Live display | `with Live(renderable) as live: live.update(...)` |
| Capture output | `with console.capture() as c: ...; text = c.get()` |
| Export to HTML | `Console(record=True)` then `console.export_html()` |
| Export to SVG | `Console(record=True)` then `console.export_svg()` |

---

## Further Reading

- **Official Documentation:** [https://rich.readthedocs.io/en/stable/](https://rich.readthedocs.io/en/stable/)
- **GitHub Repository:** [https://github.com/Textualize/rich](https://github.com/Textualize/rich)
- **PyPI Page:** [https://pypi.org/project/rich/](https://pypi.org/project/rich/)
- **Textual** (TUI framework built on Rich): [https://github.com/Textualize/textual](https://github.com/Textualize/textual)
- **Rich CLI** (command-line tool): [https://github.com/Textualize/rich-cli](https://github.com/Textualize/rich-cli)
