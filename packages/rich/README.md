# Rich

## Overview

Rich is a Python library for rendering **rich text and beautiful formatting** in the terminal. It provides a high-level API for producing styled, colorful output with minimal code. Rich can render tables, progress bars, syntax-highlighted code, tracebacks, markdown, and much more, all directly in a terminal emulator.

Created by Will McGuigan and maintained by [Textualize](https://github.com/Textualize), Rich works across Linux, macOS, and Windows (with Windows Terminal or the new Windows Console). It requires Python 3.8 or later and has minimal dependencies (only `markdown-it-py` and `pygments`).

**Key highlights:**

- Drop-in replacement for `print()` with automatic pretty-printing
- Over 16.7 million colors (truecolor), plus 256-color and standard 16-color fallback
- Tables, panels, columns, trees, progress bars, spinners, and more
- Syntax highlighting for 300+ languages via Pygments
- Markdown rendering in the terminal
- Beautiful tracebacks with local variable inspection
- Logging handler that produces gorgeous log output
- Live-updating displays and layouts
- Full emoji support

**Repository:** [https://github.com/Textualize/rich](https://github.com/Textualize/rich)
**Documentation:** [https://rich.readthedocs.io/en/stable/](https://rich.readthedocs.io/en/stable/)

---

## Latest Stable Version

As of early 2025, the latest stable release of Rich is **13.9.4** (released December 2024). Rich follows semantic versioning. Check PyPI for the most current release:

- **PyPI:** [https://pypi.org/project/rich/](https://pypi.org/project/rich/)

---

## Installation

Install from PyPI using pip:

```bash
pip install rich
```

With optional extras (for Jupyter notebook support):

```bash
pip install "rich[jupyter]"
```

Verify installation:

```bash
python -m rich
```

This runs a built-in demo that showcases Rich's capabilities. You can also check the version programmatically:

```python
import rich
print(rich.__version__)
```

---

## Core API

### `Console()` -- The Main Interface

The `Console` class is the central object in Rich. It manages output, styles, color detection, and terminal dimensions.

```python
from rich.console import Console

console = Console()
```

**Key constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `color_system` | `str \| None` | `"auto"` | Color system: `"auto"`, `"standard"` (16), `"256"`, `"truecolor"`, `"windows"`, or `None` (no color) |
| `force_terminal` | `bool \| None` | `None` | Force terminal mode (useful in CI/containers) |
| `force_jupyter` | `bool \| None` | `None` | Force Jupyter mode |
| `width` | `int \| None` | `None` | Override terminal width |
| `height` | `int \| None` | `None` | Override terminal height |
| `file` | `IO[str]` | `stderr`/`stdout` | Output file object |
| `stderr` | `bool` | `False` | Write to stderr instead of stdout |
| `record` | `bool` | `False` | Enable output recording for export |
| `markup` | `bool` | `True` | Enable console markup globally |
| `highlight` | `bool` | `True` | Enable automatic highlighting |
| `theme` | `Theme \| None` | `None` | Custom theme for styles |
| `tab_size` | `int` | `8` | Tab stop size |
| `soft_wrap` | `bool` | `False` | Enable soft wrapping (no word-boundary breaks) |

---

### `console.print()` -- Rich Printing

`console.print()` is the workhorse method. It replaces the built-in `print()` with rich text support.

```python
from rich.console import Console
console = Console()

# Basic styled output
console.print("Hello, [bold magenta]World[/bold magenta]!", ":vampire:")

# With keyword-style styling
console.print("Danger!", style="bold red")

# Multiple objects, like built-in print
console.print("Name:", "Alice", "Age:", 30)

# Automatic pretty-printing of data structures
console.print({"key": "value", "numbers": [1, 2, 3]})
```

**Key parameters for `console.print()`:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `*objects` | `Any` | -- | Objects to print (renderables, strings, data) |
| `sep` | `str` | `" "` | Separator between objects |
| `end` | `str` | `"\n"` | String appended after the last object |
| `style` | `str \| Style` | `None` | Style applied to all output |
| `justify` | `str` | `None` | Justify text: `"left"`, `"center"`, `"right"`, `"full"` |
| `overflow` | `str` | `None` | Overflow method: `"fold"`, `"crop"`, `"ellipsis"` |
| `highlight` | `bool` | `None` | Enable/disable highlighting for this call |
| `markup` | `bool` | `None` | Enable/disable markup for this call |
| `width` | `int` | `None` | Override width for this call |
| `soft_wrap` | `bool` | `False` | Enable soft wrapping |
| `new_line_start` | `bool` | `False` | Insert newline before output if not at start of line |

---

### `console.log()` -- Logging with Timestamps

`console.log()` works like `console.print()` but prepends an automatic timestamp and the caller's file/line number. Useful for debug-style logging.

```python
console.log("Server started", log_locals=True)
console.log("[bold red]Error:[/] Connection failed")
```

**Additional parameters (beyond those shared with `print()`):**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `log_locals` | `bool` | `False` | Display a table of local variables |
| `_stack_offset` | `int` | `1` | Stack offset for caller info |

---

### `console.rule()` -- Horizontal Rule

Draws a horizontal line across the terminal, optionally with a centered title.

```python
console.rule("Chapter 1")
console.rule()  # plain line
console.rule("[bold blue]Section", style="blue", align="left")
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `title` | `str` | `""` | Text to display in the rule |
| `characters` | `str` | `"-"` | Character(s) used to draw the line |
| `style` | `str \| Style` | `"rule.line"` | Style for the line |
| `align` | `str` | `"center"` | Title alignment: `"left"`, `"center"`, `"right"` |

---

### `console.status()` -- Spinner Context Manager

Displays a spinner animation while a block of code executes. The spinner automatically stops when the context manager exits.

```python
with console.status("Working...", spinner="dots"):
    import time
    time.sleep(3)

# Or with a custom spinner
with console.status("[bold green]Downloading files...", spinner="bouncingBar"):
    do_work()
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | `str \| RenderableType` | -- | Status message to display |
| `spinner` | `str` | `"dots"` | Name of spinner animation |
| `spinner_style` | `str \| Style` | `"status.spinner"` | Style for the spinner |
| `speed` | `float` | `1.0` | Speed multiplier for the spinner |
| `refresh_per_second` | `float` | `12.5` | Refresh rate for the animation |

Available spinners include: `"dots"`, `"line"`, `"dots2"`, `"dots3"`, `"bouncingBar"`, `"arc"`, `"arrow"`, `"balloon"`, `"clock"`, `"earth"`, `"moon"`, `"pong"`, `"runner"`, `"toggle"`, and many more (inherited from the cli-spinners project).

---

### Rich Markup

Rich supports an inline markup language within strings passed to `console.print()` and other output methods. The syntax uses square brackets.

**Basic markup:**

```python
console.print("[bold]Bold text[/bold]")
console.print("[italic]Italic text[/italic]")
console.print("[underline]Underlined[/underline]")
console.print("[strikethrough]Struck through[/strikethrough]")
console.print("[bold italic]Bold and italic[/bold italic]")
```

**Color markup:**

```python
console.print("[red]Red text[/red]")
console.print("[green]Green text[/green]")
console.print("[blue on white]Blue text on white background[/blue on white]")
console.print("[#ff8800]Custom hex color[/#ff8800]")
console.print("[rgb(100,200,50)]RGB color[/rgb(100,200,50)]")
console.print("[color(208)]256-color palette[/color(208)]")
```

**Links:**

```python
console.print("[link=https://www.python.org]Click here[/link]")
```

**Escaping markup:**

```python
# Use double brackets to escape
console.print("Use \\[bold] for bold markup")
# Or disable markup for the call
console.print("[this is not markup]", markup=False)
```

**Closing tags:**

```python
# Explicit close
console.print("[bold red]Important[/bold red]")

# Close all with [/]
console.print("[bold red]Important[/]")

# Tags are nested
console.print("[bold]Bold [red]and red[/red] just bold[/bold]")
```

---

## Components

### `Table` -- Configurable Tables

Rich tables support column alignment, styles, borders, padding, row/column separators, and more.

```python
from rich.table import Table
from rich.console import Console

console = Console()

table = Table(title="Star Wars Movies")

table.add_column("Released", justify="right", style="cyan", no_wrap=True)
table.add_column("Title", style="magenta")
table.add_column("Box Office", justify="right", style="green")

table.add_row("Dec 20, 2019", "Star Wars: The Rise of Skywalker", "$952,110,690")
table.add_row("May 25, 2018", "Solo: A Star Wars Story", "$393,151,347")
table.add_row("Dec 15, 2017", "Star Wars: The Last Jedi", "$1,332,539,889")

console.print(table)
```

**`Table` constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `title` | `str \| None` | `None` | Table title (displayed above) |
| `caption` | `str \| None` | `None` | Table caption (displayed below) |
| `width` | `int \| None` | `None` | Fixed width; `None` for auto |
| `min_width` | `int \| None` | `None` | Minimum table width |
| `box` | `Box \| None` | `HEAVY_HEAD` | Border style (from `rich.box`) |
| `show_header` | `bool` | `True` | Show column headers |
| `show_footer` | `bool` | `False` | Show column footers |
| `show_edge` | `bool` | `True` | Show table border |
| `show_lines` | `bool` | `False` | Show lines between rows |
| `pad_edge` | `bool` | `True` | Pad left/right edges |
| `expand` | `bool` | `False` | Expand table to fit terminal width |
| `row_styles` | `list[str]` | `None` | Alternating row styles (e.g., `["", "dim"]`) |
| `border_style` | `str \| Style` | `None` | Style for borders |
| `header_style` | `str \| Style` | `"table.header"` | Style for headers |
| `title_style` | `str \| Style` | `"table.title"` | Style for the title |
| `padding` | `tuple \| int` | `(0, 1)` | Cell padding (top, right, bottom, left) |
| `collapse_padding` | `bool` | `False` | Collapse adjacent padding |
| `leading` | `int` | `0` | Extra lines between rows |
| `highlight` | `bool` | `False` | Highlight cell content |

**`add_column()` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `header` | `str` | `""` | Column header text |
| `footer` | `str` | `""` | Column footer text |
| `style` | `str \| Style` | `None` | Style for column cells |
| `justify` | `str` | `"left"` | `"left"`, `"center"`, `"right"`, `"full"` |
| `vertical` | `str` | `"top"` | `"top"`, `"middle"`, `"bottom"` |
| `width` | `int \| None` | `None` | Fixed column width |
| `min_width` | `int \| None` | `None` | Minimum column width |
| `max_width` | `int \| None` | `None` | Maximum column width |
| `ratio` | `int \| None` | `None` | Flexible width ratio |
| `no_wrap` | `bool` | `False` | Disable wrapping in this column |
| `overflow` | `str` | `"ellipsis"` | Overflow method |

**Box styles** (import from `rich.box`):

`ASCII`, `ASCII2`, `ASCII_DOUBLE_HEAD`, `DOUBLE`, `DOUBLE_EDGE`, `HEAVY`, `HEAVY_EDGE`, `HEAVY_HEAD`, `HORIZONTALS`, `MARKDOWN`, `MINIMAL`, `MINIMAL_DOUBLE_HEAD`, `MINIMAL_HEAVY_HEAD`, `ROUNDED`, `SIMPLE`, `SIMPLE_HEAD`, `SIMPLE_HEAVY`, `SQUARE`, `SQUARE_DOUBLE_HEAD`

```python
from rich import box
table = Table(box=box.ROUNDED)
```

---

### `Panel` -- Boxed Content

Panels wrap content in a bordered box with an optional title and subtitle.

```python
from rich.panel import Panel
from rich.console import Console

console = Console()

console.print(Panel("Hello, World!", title="Greeting", subtitle="v1.0"))
console.print(Panel.fit("Fitted panel", border_style="green"))
```

**Key parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `renderable` | `RenderableType` | -- | Content to display inside the panel |
| `title` | `str \| None` | `None` | Title displayed at the top border |
| `subtitle` | `str \| None` | `None` | Subtitle displayed at the bottom border |
| `title_align` | `str` | `"center"` | `"left"`, `"center"`, `"right"` |
| `subtitle_align` | `str` | `"center"` | `"left"`, `"center"`, `"right"` |
| `box` | `Box` | `ROUNDED` | Box style |
| `border_style` | `str \| Style` | `"none"` | Style for the border |
| `width` | `int \| None` | `None` | Fixed width |
| `height` | `int \| None` | `None` | Fixed height |
| `padding` | `tuple \| int` | `(0, 1)` | Internal padding |
| `expand` | `bool` | `True` | Expand to fill available width |
| `highlight` | `bool` | `False` | Highlight content |

Use `Panel.fit()` as a class method to create a panel that fits its content rather than expanding.

---

### `Columns` -- Multi-Column Layout

Renders an iterable of renderables in multiple columns that fill the terminal width.

```python
from rich.columns import Columns
from rich.panel import Panel
from rich.console import Console

console = Console()

renderables = [Panel(f"Item {i}", expand=True) for i in range(12)]
console.print(Columns(renderables))
```

**Key parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `renderables` | `Iterable` | -- | Items to arrange in columns |
| `padding` | `tuple \| int` | `(0, 1)` | Padding between columns |
| `expand` | `bool` | `False` | Expand to fill width |
| `equal` | `bool` | `False` | Make all columns equal width |
| `column_first` | `bool` | `False` | Fill columns top-to-bottom first |
| `width` | `int \| None` | `None` | Fixed column width |
| `title` | `str \| None` | `None` | Title displayed above columns |

---

### `Tree` -- Tree Display

Renders hierarchical data as a tree with guide lines.

```python
from rich.tree import Tree
from rich.console import Console

console = Console()

tree = Tree("Root")
branch1 = tree.add("Branch 1")
branch1.add("Leaf 1a")
branch1.add("Leaf 1b")
branch2 = tree.add("Branch 2")
branch2.add("[red]Leaf 2a[/red]")
sub = branch2.add("Sub-branch")
sub.add("Deep leaf")

console.print(tree)
```

**`Tree` constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `label` | `str \| RenderableType` | -- | Root label |
| `style` | `str \| Style` | `"tree"` | Style for guide lines |
| `guide_style` | `str \| Style` | `"tree.line"` | Style for the tree guides |
| `expanded` | `bool` | `True` | Whether the tree starts expanded |
| `highlight` | `bool` | `False` | Highlight labels |

**`tree.add()` returns a new `Tree` node**, so you can chain and build deeply nested trees.

---

### `Syntax` -- Syntax-Highlighted Code

Renders source code with syntax highlighting powered by Pygments.

```python
from rich.syntax import Syntax
from rich.console import Console

console = Console()

code = '''\
def fibonacci(n):
    """Generate the first n Fibonacci numbers."""
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b
'''

syntax = Syntax(code, "python", theme="monokai", line_numbers=True)
console.print(syntax)

# From a file
syntax = Syntax.from_path("example.py", theme="dracula", line_numbers=True)
console.print(syntax)
```

**Key parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `code` | `str` | -- | Source code string |
| `lexer` | `str` | -- | Pygments lexer name (e.g., `"python"`, `"javascript"`, `"rust"`) |
| `theme` | `str` | `"monokai"` | Pygments style theme |
| `line_numbers` | `bool` | `False` | Show line numbers |
| `start_line` | `int` | `1` | Starting line number |
| `line_range` | `tuple \| None` | `None` | Range of lines to display, e.g., `(5, 15)` |
| `highlight_lines` | `set[int]` | `None` | Set of line numbers to highlight |
| `word_wrap` | `bool` | `False` | Wrap long lines |
| `code_width` | `int \| None` | `None` | Fixed width for code area |
| `dedent` | `bool` | `False` | Dedent code before rendering |
| `padding` | `int` | `0` | Padding around the code |
| `background_color` | `str \| None` | `None` | Override background color |

**Available themes:** `"monokai"`, `"dracula"`, `"github-dark"`, `"one-dark"`, `"vim"`, `"native"`, `"emacs"`, `"paraiso-dark"`, `"solarized-dark"`, `"solarized-light"`, `"zenburn"`, and any Pygments style name.

---

### `Markdown` -- Rendered Markdown

Renders Markdown text in the terminal with headings, lists, code blocks, blockquotes, links, and more.

```python
from rich.markdown import Markdown
from rich.console import Console

console = Console()

md = Markdown("""\
# Heading 1

This is **bold** and *italic* text.

- Item 1
- Item 2
  - Nested item

```python
print("code block")
```

> A blockquote

| Col 1 | Col 2 |
|-------|-------|
| A     | B     |
""")

console.print(md)
```

**Key parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `markup` | `str` | -- | Markdown text |
| `code_theme` | `str` | `"monokai"` | Theme for code blocks |
| `inline_code_lexer` | `str \| None` | `None` | Default lexer for inline code |
| `inline_code_theme` | `str \| None` | `None` | Theme for inline code |
| `justify` | `str \| None` | `None` | Text justification |
| `hyperlinks` | `bool` | `True` | Render links as terminal hyperlinks |

---

### `Progress` -- Progress Bars

Rich provides elegant, configurable progress bars for tracking long-running tasks.

**Simple usage with `track()`:**

```python
from rich.progress import track
import time

for step in track(range(100), description="Processing..."):
    time.sleep(0.02)
```

**Advanced multi-task progress:**

```python
from rich.progress import Progress

with Progress() as progress:
    task1 = progress.add_task("[red]Downloading...", total=1000)
    task2 = progress.add_task("[green]Processing...", total=1000)
    task3 = progress.add_task("[cyan]Uploading...", total=1000)

    while not progress.finished:
        progress.update(task1, advance=0.5)
        progress.update(task2, advance=0.3)
        progress.update(task3, advance=0.9)
        import time
        time.sleep(0.02)
```

**Customizing columns:**

```python
from rich.progress import (
    Progress,
    BarColumn,
    TextColumn,
    TimeRemainingColumn,
    TimeElapsedColumn,
    SpinnerColumn,
    MofNCompleteColumn,
    TaskProgressColumn,
    DownloadColumn,
    TransferSpeedColumn,
)

progress = Progress(
    SpinnerColumn(),
    TextColumn("[bold blue]{task.description}"),
    BarColumn(bar_width=40),
    TaskProgressColumn(),
    TimeRemainingColumn(),
    TimeElapsedColumn(),
)
```

**Available built-in columns:**

| Column Class | Description |
|-------------|-------------|
| `TextColumn` | Arbitrary text (supports `{task.fields}` format) |
| `BarColumn` | The progress bar itself |
| `TaskProgressColumn` | Shows percentage (e.g., `45%`) |
| `TimeRemainingColumn` | Estimated time remaining |
| `TimeElapsedColumn` | Elapsed time |
| `MofNCompleteColumn` | Shows `M/N` completed |
| `SpinnerColumn` | A spinner animation |
| `DownloadColumn` | Shows download size (e.g., `5.2/10.0 MB`) |
| `TransferSpeedColumn` | Shows transfer speed |
| `FileSizeColumn` | Shows file size |
| `TotalFileSizeColumn` | Shows total file size |
| `RenderableColumn` | Renders arbitrary renderables |

**Indeterminate progress:**

```python
# Omit total for an indeterminate (pulsing) progress bar
task = progress.add_task("Searching...", total=None)
```

---

### `Prompt` / `Confirm` -- User Input

Rich provides styled prompts for user input.

```python
from rich.prompt import Prompt, Confirm, IntPrompt, FloatPrompt

# Basic prompt
name = Prompt.ask("What is your name?")

# With default value
name = Prompt.ask("What is your name?", default="World")

# With choices (validates input)
color = Prompt.ask("Pick a color", choices=["red", "green", "blue"])

# Password input (hidden)
password = Prompt.ask("Enter password", password=True)

# Integer prompt (validates and converts)
age = IntPrompt.ask("How old are you?")

# Float prompt
weight = FloatPrompt.ask("Enter weight in kg")

# Yes/no confirmation
if Confirm.ask("Do you want to continue?"):
    print("Continuing...")
```

**`Prompt.ask()` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `prompt` | `str` | -- | The prompt text |
| `default` | `Any` | `...` | Default value (shown in brackets) |
| `choices` | `list[str] \| None` | `None` | Valid choices |
| `password` | `bool` | `False` | Hide input |
| `show_default` | `bool` | `True` | Show the default value |
| `show_choices` | `bool` | `True` | Show valid choices |
| `console` | `Console \| None` | `None` | Console to use |

---

### `Pretty` -- Auto-Formatting Data Structures

Automatically formats Python data structures (dicts, lists, sets, tuples, etc.) with indentation and syntax coloring.

```python
from rich.pretty import Pretty, pprint, pretty_repr, install
from rich.console import Console

console = Console()

data = {"name": "Alice", "scores": [98, 87, 92], "active": True}

# As a renderable
console.print(Pretty(data))

# Convenience function (standalone)
pprint(data)

# Install as the default repr for the REPL
install()  # Now all REPL output is pretty-printed

# Get a pretty string
s = pretty_repr(data, max_width=60)
```

**`Pretty` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `_object` | `Any` | -- | Object to format |
| `highlighter` | `HighlighterType \| None` | `None` | Highlighter to apply |
| `indent_size` | `int` | `4` | Indentation size |
| `justify` | `str \| None` | `None` | Text justification |
| `overflow` | `str \| None` | `None` | Overflow mode |
| `no_wrap` | `bool` | `False` | Disable wrapping |
| `indent_guides` | `bool` | `False` | Show indent guides |
| `max_length` | `int \| None` | `None` | Max items before truncation |
| `max_string` | `int \| None` | `None` | Max string length before truncation |
| `expand_all` | `bool` | `False` | Always expand containers |

---

### `Traceback` -- Beautiful Tracebacks

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

### `Layout` -- Split-Screen Terminal Layouts

Split the terminal into multiple regions (horizontally or vertically).

```python
from rich.layout import Layout
from rich.panel import Panel
from rich.console import Console

console = Console()

layout = Layout()

# Split into rows
layout.split_column(
    Layout(name="header", size=3),
    Layout(name="body", ratio=1),
    Layout(name="footer", size=3),
)

# Split body into columns
layout["body"].split_row(
    Layout(name="left"),
    Layout(name="right"),
)

# Assign content to regions
layout["header"].update(Panel("Header"))
layout["left"].update(Panel("Left Panel"))
layout["right"].update(Panel("Right Panel"))
layout["footer"].update(Panel("Footer"))

console.print(layout)
```

**Key methods:**

| Method | Description |
|--------|-------------|
| `split_column(*layouts)` | Split vertically into rows |
| `split_row(*layouts)` | Split horizontally into columns |
| `update(renderable)` | Set content for a layout region |
| `unsplit()` | Remove child layouts |

**Layout constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str \| None` | `None` | Name for referencing via `layout["name"]` |
| `size` | `int \| None` | `None` | Fixed size in lines/columns |
| `minimum_size` | `int` | `1` | Minimum size |
| `ratio` | `int` | `1` | Flexible ratio (like CSS flex) |
| `visible` | `bool` | `True` | Whether the layout is visible |
| `renderable` | `RenderableType` | `None` | Initial content |

---

### `Live` -- Live-Updating Displays

`Live` provides a context manager that continuously updates a renderable on screen, replacing the previous output.

```python
from rich.live import Live
from rich.table import Table
from rich.console import Console
import time

console = Console()

def generate_table(row_count: int) -> Table:
    table = Table()
    table.add_column("ID")
    table.add_column("Value")
    for i in range(row_count):
        table.add_row(str(i), f"[green]{i * i}[/green]")
    return table

with Live(generate_table(0), console=console, refresh_per_second=4) as live:
    for i in range(1, 20):
        time.sleep(0.3)
        live.update(generate_table(i))
```

**`Live` constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `renderable` | `RenderableType` | `None` | Initial renderable to display |
| `console` | `Console \| None` | `None` | Console instance |
| `screen` | `bool` | `False` | Use full-screen mode (alternate screen) |
| `auto_refresh` | `bool` | `True` | Automatically refresh display |
| `refresh_per_second` | `float` | `4` | Refresh rate |
| `transient` | `bool` | `False` | Clear display when context manager exits |
| `redirect_stdout` | `bool` | `True` | Redirect stdout through Live |
| `redirect_stderr` | `bool` | `True` | Redirect stderr through Live |
| `vertical_overflow` | `str` | `"ellipsis"` | `"crop"`, `"ellipsis"`, `"visible"` |
| `get_renderable` | `Callable \| None` | `None` | Callback to get renderable (alternative to `update()`) |

---

## Styling System

### Style Strings

Rich uses a compact string syntax for defining styles. Multiple attributes are combined with spaces.

```python
from rich.console import Console
console = Console()

console.print("Hello", style="bold")
console.print("Hello", style="bold red")
console.print("Hello", style="bold red on white")
console.print("Hello", style="bold italic underline red on #1a1a2e")
console.print("Hello", style="not bold")  # explicitly disable bold
console.print("Hello", style="link https://example.com")
```

**Style string components:**

| Component | Examples | Description |
|-----------|----------|-------------|
| **Attributes** | `bold`, `dim`, `italic`, `underline`, `blink`, `blink2`, `reverse`, `conceal`, `strike`, `overline` | Text decoration |
| **Negation** | `not bold`, `not italic` | Explicitly disable attribute |
| **Foreground** | `red`, `bright_red`, `#ff0000`, `rgb(255,0,0)`, `color(196)` | Text color |
| **Background** | `on red`, `on #ff0000`, `on rgb(255,0,0)` | Background color |
| **Link** | `link https://...` | Clickable terminal hyperlink |

**Named colors:** `black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`, and their `bright_*` variants. Plus all CSS color names (`salmon`, `gold`, `dodger_blue`, `medium_spring_green`, etc.).

---

### `Style` Objects

For programmatic style creation and composition:

```python
from rich.style import Style

# Create a style
warning_style = Style(bold=True, color="red", bgcolor="yellow")

# Parse from string
info_style = Style.parse("italic cyan")

# Combine styles (right style overrides left)
combined = Style(bold=True) + Style(color="red")

# Use in print
console.print("Warning!", style=warning_style)
```

**`Style` constructor parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `color` | `str \| Color \| None` | `None` | Foreground color |
| `bgcolor` | `str \| Color \| None` | `None` | Background color |
| `bold` | `bool \| None` | `None` | Bold text |
| `dim` | `bool \| None` | `None` | Dim text |
| `italic` | `bool \| None` | `None` | Italic text |
| `underline` | `bool \| None` | `None` | Underlined text |
| `blink` | `bool \| None` | `None` | Blinking text |
| `blink2` | `bool \| None` | `None` | Fast blinking text |
| `reverse` | `bool \| None` | `None` | Reverse foreground/background |
| `conceal` | `bool \| None` | `None` | Concealed text |
| `strike` | `bool \| None` | `None` | Strikethrough text |
| `overline` | `bool \| None` | `None` | Overlined text |
| `link` | `str \| None` | `None` | URL for terminal hyperlink |

---

### `Theme` -- Consistent Styling

Themes let you define named styles and apply them consistently across your application.

```python
from rich.theme import Theme
from rich.console import Console

custom_theme = Theme({
    "info": "dim cyan",
    "warning": "bold yellow",
    "error": "bold red",
    "success": "bold green",
    "heading": "bold underline magenta",
    "repr.number": "bold cyan",
})

console = Console(theme=custom_theme)

console.print("Server started", style="info")
console.print("Low disk space", style="warning")
console.print("Connection failed", style="error")
console.print("All tests passed", style="success")

# Use theme styles in markup
console.print("[info]Informational message[/info]")
console.print("[error]Error message[/error]")
```

**Loading themes from files:**

```ini
# theme.ini
[styles]
info = dim cyan
warning = bold yellow
error = bold red
success = bold green
```

```python
from rich.theme import Theme

theme = Theme.read("theme.ini")
console = Console(theme=theme)
```

**Overriding built-in styles:**

Rich uses many built-in style names (like `"repr.number"`, `"json.key"`, `"markdown.h1"`, `"progress.bar.complete"`, etc.). You can override any of them via a custom theme.

---

### Console Markup Syntax

Console markup is Rich's inline styling language, used within strings:

```python
console.print("[bold red]Error:[/bold red] Something went wrong")
console.print("[on blue] highlighted [/on blue]")
console.print("[bold]Bold [italic]Bold+Italic[/italic] Bold[/bold]")
console.print("[link=https://example.com]Clickable text[/link]")

# Using theme style names in markup
console.print("[error]This uses the 'error' theme style[/error]")

# Emoji shortcodes
console.print(":thumbs_up: :rocket: :warning:")
```

Markup is processed by default in `console.print()`. Disable it with `markup=False` or at the console level with `Console(markup=False)`.

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

## Integration

### `RichHandler` -- As a Logging Handler

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

### With Click / Typer

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

### `rich.traceback.install()` -- Default Traceback Handler

Install Rich as the default traceback handler for all uncaught exceptions:

```python
from rich.traceback import install
install(show_locals=True, suppress=["click"])

# Now any uncaught exception will be rendered with Rich
```

This sets `sys.excepthook` to use Rich's traceback renderer. It is recommended to call this at application startup. The `suppress` parameter lets you hide frames from third-party libraries to reduce noise.

---

## Gotchas and Common Mistakes

### 1. Console Width Detection in CI/Containers

**Problem:** In CI environments, Docker containers, or when piping output, Rich may not detect the terminal width correctly. It may default to 80 columns or produce no color output.

**Solution:**

```python
import os

# Option 1: Set environment variable
os.environ["COLUMNS"] = "120"

# Option 2: Force terminal mode and set width explicitly
console = Console(force_terminal=True, width=120)

# Option 3: Environment variable for color
os.environ["FORCE_COLOR"] = "1"
```

In CI systems (GitHub Actions, GitLab CI), set `COLUMNS` and `FORCE_COLOR` in your workflow configuration.

---

### 2. Capturing Output (StringIO vs Console(file=...))

**Problem:** Using `io.StringIO` with Rich does not work as expected because Rich writes ANSI escape codes.

**Solution:**

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

### 3. Color Support Detection

**Problem:** Rich auto-detects color support but may get it wrong in some environments (e.g., screen/tmux, SSH sessions, or unusual terminal emulators).

**Solution:**

```python
# Force a specific color system
console = Console(color_system="truecolor")  # Force 16.7M colors
console = Console(color_system="256")         # Force 256 colors
console = Console(color_system="standard")    # Force 16 colors
console = Console(color_system=None)          # Disable all colors

# Check what Rich detected
console = Console()
print(console.color_system)    # "truecolor", "256", "standard", etc.
print(console.is_terminal)     # True if connected to a terminal
```

The `COLORTERM` environment variable (`"truecolor"` or `"24bit"`) is used by Rich for truecolor detection. You can also set `NO_COLOR=1` to disable all color output (Rich respects the [NO_COLOR](https://no-color.org/) convention).

---

### 4. Performance with Very Large Outputs

**Problem:** Rendering very large tables (thousands of rows) or printing massive amounts of styled text can be slow because Rich builds an internal representation before writing.

**Solution:**

```python
# For large tables, consider pagination
# Print a subset rather than everything at once

# Disable features you don't need
console = Console(highlight=False, markup=False)

# For progress bars, reduce refresh rate
with Progress(refresh_per_second=2) as progress:
    ...

# For very large files, use Syntax with line_range
syntax = Syntax.from_path("huge_file.py", line_range=(100, 200))
```

---

### 5. Thread Safety with Console

**Problem:** `Console` is **not** fully thread-safe. Multiple threads writing to the same Console can produce interleaved or garbled output.

**Solution:**

```python
import threading
from rich.console import Console

console = Console()
lock = threading.Lock()

def thread_safe_print(message: str):
    with lock:
        console.print(message)

# Or use separate Console instances per thread
# (output may still interleave at the terminal level)

# For progress bars, Rich's Progress class handles
# thread safety internally for task updates
from rich.progress import Progress

with Progress() as progress:
    task = progress.add_task("Working...", total=100)
    # Safe to call from multiple threads:
    progress.update(task, advance=1)
```

Note: `Progress`, `Live`, and `Status` use internal locking and are safe for multi-threaded task updates, but you should not print to the same Console from other threads while they are active.

---

### 6. Markup in f-strings with User Data

**Problem:** User-provided data in f-strings can break Rich markup if it contains square brackets.

**Solution:**

```python
from rich.markup import escape

user_input = "my [bold] data"

# WRONG: will be interpreted as markup
console.print(f"User said: {user_input}")

# RIGHT: escape user data
console.print(f"User said: {escape(user_input)}")

# Or disable markup for the call
console.print(f"User said: {user_input}", markup=False)
```

---

## Complete Code Examples

### Example 1: Styled Console Output

```python
from rich.console import Console
from rich.panel import Panel
from rich.text import Text

console = Console()

# Basic styled printing
console.print("[bold cyan]Welcome[/bold cyan] to the [italic]Rich[/italic] demo!")

# Using style parameter
console.print("This is an error message.", style="bold red")
console.print("This is a success message.", style="bold green")

# Justified text
console.print("Left aligned", justify="left")
console.print("Center aligned", justify="center")
console.print("Right aligned", justify="right")

# Rule separator
console.rule("[bold blue]Section Title")

# Panel with styled content
text = Text.assemble(
    ("Status: ", "bold"),
    ("Online", "bold green"),
    " | ",
    ("Uptime: ", "bold"),
    ("47 days", "cyan"),
)
console.print(Panel(text, title="Server Info", border_style="green"))

# Log-style output
console.log("Application initialized")
console.log("Connected to database", log_locals=False)
```

---

### Example 2: Building Tables

```python
from rich.console import Console
from rich.table import Table
from rich import box

console = Console()

# Basic table
table = Table(
    title="Quarterly Report",
    caption="All figures in USD",
    box=box.ROUNDED,
    show_lines=True,
    title_style="bold magenta",
    border_style="bright_blue",
    row_styles=["", "dim"],  # alternating row styles
)

table.add_column("Quarter", justify="center", style="cyan", no_wrap=True)
table.add_column("Revenue", justify="right", style="green")
table.add_column("Expenses", justify="right", style="red")
table.add_column("Profit", justify="right", style="bold green")

table.add_row("Q1 2024", "$1,234,567", "$987,654", "$246,913")
table.add_row("Q2 2024", "$1,345,678", "$1,023,456", "$322,222")
table.add_row("Q3 2024", "$1,456,789", "$1,098,765", "$358,024")
table.add_row("Q4 2024", "$1,567,890", "$1,134,567", "$433,323")

# Add a footer
table.columns[1].footer = "$5,604,924"
table.columns[2].footer = "$4,244,442"
table.columns[3].footer = "$1,360,482"
table.columns[0].footer = "Total"
table.show_footer = True

console.print(table)

# Nested table (table inside a table)
inner = Table(box=box.SIMPLE)
inner.add_column("Metric")
inner.add_column("Value")
inner.add_row("CPU", "45%")
inner.add_row("Memory", "2.3 GB")

outer = Table(title="Dashboard")
outer.add_column("Server")
outer.add_column("Details")
outer.add_row("web-01", inner)

console.print(outer)
```

---

### Example 3: Progress Bars for Downloads

```python
import time
from rich.progress import (
    Progress,
    BarColumn,
    DownloadColumn,
    TextColumn,
    TransferSpeedColumn,
    TimeRemainingColumn,
    SpinnerColumn,
)
from rich.console import Console

console = Console()

# Simulated download progress
files = [
    ("document.pdf", 15_000_000),
    ("image.png", 5_500_000),
    ("video.mp4", 150_000_000),
    ("archive.zip", 45_000_000),
]

progress = Progress(
    SpinnerColumn(),
    TextColumn("[bold blue]{task.description}", justify="right"),
    BarColumn(bar_width=40),
    "[progress.percentage]{task.percentage:>3.1f}%",
    DownloadColumn(),
    TransferSpeedColumn(),
    TimeRemainingColumn(),
    console=console,
)

with progress:
    tasks = []
    for filename, total in files:
        task_id = progress.add_task(filename, total=total)
        tasks.append((task_id, total))

    # Simulate concurrent downloads
    while not progress.finished:
        for task_id, total in tasks:
            # Simulate varying speeds
            chunk = min(total * 0.02, 1_000_000)
            progress.update(task_id, advance=chunk)
        time.sleep(0.05)

console.print("[bold green]All downloads complete!")
```

---

### Example 4: Live Updating Display

```python
import time
import random
from rich.live import Live
from rich.table import Table
from rich.console import Console
from rich.panel import Panel
from rich.layout import Layout
from rich.text import Text

console = Console()

def make_dashboard(iteration: int) -> Layout:
    """Build a dashboard layout that updates each iteration."""
    layout = Layout()
    layout.split_column(
        Layout(name="header", size=3),
        Layout(name="body", ratio=1),
    )
    layout["body"].split_row(
        Layout(name="stats"),
        Layout(name="log"),
    )

    # Header
    header_text = Text(f"Live Dashboard (update #{iteration})", style="bold white on blue", justify="center")
    layout["header"].update(Panel(header_text, style="blue"))

    # Stats table
    stats = Table(title="System Metrics", expand=True)
    stats.add_column("Metric", style="cyan")
    stats.add_column("Value", justify="right", style="green")
    stats.add_row("CPU Usage", f"{random.randint(20, 95)}%")
    stats.add_row("Memory", f"{random.uniform(1.5, 7.8):.1f} GB")
    stats.add_row("Disk I/O", f"{random.randint(10, 500)} MB/s")
    stats.add_row("Network", f"{random.randint(1, 100)} Mbps")
    stats.add_row("Requests/s", f"{random.randint(100, 5000)}")
    layout["stats"].update(Panel(stats))

    # Log panel
    log_lines = []
    levels = ["[green]INFO[/green]", "[yellow]WARN[/yellow]", "[red]ERROR[/red]"]
    messages = ["Request handled", "Cache miss", "Connection timeout", "Query executed", "User login"]
    for _ in range(8):
        level = random.choice(levels)
        msg = random.choice(messages)
        log_lines.append(f"  {level}  {msg}")
    layout["log"].update(Panel("\n".join(log_lines), title="Recent Logs"))

    return layout

with Live(make_dashboard(0), console=console, refresh_per_second=2, screen=False) as live:
    for i in range(1, 30):
        time.sleep(0.5)
        live.update(make_dashboard(i))
```

---

### Example 5: Custom Themes

```python
from rich.console import Console
from rich.theme import Theme
from rich.panel import Panel
from rich.table import Table

# Define a custom theme
my_theme = Theme({
    "info": "dim cyan",
    "warning": "bold yellow",
    "error": "bold white on red",
    "success": "bold green",
    "heading": "bold underline magenta",
    "key": "bold cyan",
    "value": "italic bright_white",
    "muted": "dim white",
    "highlight": "bold yellow on #333333",
    "repr.number": "bold bright_cyan",
    "repr.str": "bright_green",
    "bar.complete": "bright_green",
    "bar.finished": "bold bright_green",
    "progress.percentage": "bold cyan",
})

console = Console(theme=my_theme)

# Use theme styles in markup
console.print("[heading]Application Status Report[/heading]")
console.print()

console.print("[info]All systems operational[/info]")
console.print("[warning]High memory usage detected on node-03[/warning]")
console.print("[error] CRITICAL: Database replica lag exceeding 30s [/error]")
console.print("[success]Deployment v2.4.1 completed successfully[/success]")

console.print()
console.rule("[heading]Configuration")

# Theme styles in tables
table = Table(title="Environment Variables", border_style="bright_blue")
table.add_column("Key", style="key")
table.add_column("Value", style="value")
table.add_column("Status", justify="center")

table.add_row("DATABASE_URL", "postgres://db:5432/app", "[success]Set[/success]")
table.add_row("REDIS_URL", "redis://cache:6379", "[success]Set[/success]")
table.add_row("SECRET_KEY", "********", "[success]Set[/success]")
table.add_row("DEBUG", "true", "[warning]Review[/warning]")
table.add_row("SENTRY_DSN", "", "[error]Missing[/error]")

console.print(table)
```

---

### Example 6: Logging with RichHandler

```python
import logging
from rich.logging import RichHandler
from rich.console import Console

# Create a console (optional, for customization)
console = Console(stderr=True)

# Configure logging with RichHandler
logging.basicConfig(
    level=logging.DEBUG,
    format="%(message)s",
    datefmt="[%X]",
    handlers=[
        RichHandler(
            console=console,
            rich_tracebacks=True,
            tracebacks_show_locals=True,
            tracebacks_suppress=["urllib3", "requests"],
            markup=True,
            show_path=True,
            enable_link_path=True,
        )
    ],
)

log = logging.getLogger("myapp")

# Basic logging
log.debug("Initializing application components")
log.info("Server started on [bold cyan]http://0.0.0.0:8000[/bold cyan]")
log.warning("Cache directory not found, creating [italic]/tmp/cache[/italic]")
log.error("Failed to connect to database after 3 retries")
log.critical("[bold white on red]SYSTEM FAILURE: Out of memory[/bold white on red]")

# Logging with extra data
log.info("User authenticated", extra={"markup": True})

# Exception logging with rich tracebacks
try:
    result = 1 / 0
except Exception:
    log.exception("Calculation failed")

# Using named loggers
db_log = logging.getLogger("myapp.database")
db_log.info("Connection pool initialized (size=10)")

api_log = logging.getLogger("myapp.api")
api_log.info("Registered 42 routes")
```

---

### Example 7: Pretty Tracebacks

```python
from rich.traceback import install
from rich.console import Console

# Install rich tracebacks globally
install(
    show_locals=True,
    width=120,
    extra_lines=5,
    theme="monokai",
    word_wrap=True,
    suppress=["click", "uvicorn"],  # Hide frames from these libraries
)


# Example: nested exception with context
def fetch_user(user_id: int) -> dict:
    users = {"1": "Alice", "2": "Bob"}
    return users[str(user_id)]


def process_request(request: dict) -> str:
    user_id = request["user_id"]
    user = fetch_user(user_id)
    return f"Hello, {user}"


def handle_api_call():
    request = {"user_id": 999, "action": "greet"}
    try:
        result = process_request(request)
    except KeyError as e:
        raise ValueError(f"User not found: {e}") from e


# This will produce a beautiful traceback with:
# - Syntax-highlighted source code
# - Local variables in each stack frame
# - Chained exception display
# - Clean, readable formatting
try:
    handle_api_call()
except Exception:
    console = Console()
    console.print_exception(show_locals=True)
```

---

### Example 8: Rich Inspect

```python
import rich
from rich.console import Console

console = Console()

# Inspect a built-in type
rich.inspect(str, methods=True)

# Inspect a module
import os.path
rich.inspect(os.path, help=True)

# Inspect an instance with its methods
my_list = [1, 2, 3]
rich.inspect(my_list, methods=True, docs=True)

# Inspect a custom class
class Server:
    """A simple server class for demonstration."""

    def __init__(self, host: str = "localhost", port: int = 8000):
        self.host = host
        self.port = port
        self._connections = []
        self.is_running = False

    def start(self) -> None:
        """Start the server."""
        self.is_running = True

    def stop(self) -> None:
        """Stop the server gracefully."""
        self.is_running = False

    @property
    def address(self) -> str:
        """Full server address."""
        return f"{self.host}:{self.port}"

server = Server("0.0.0.0", 9000)
rich.inspect(server, methods=True, help=True)
```

---

### Example 9: Combining Components (Full Application)

```python
"""A complete Rich demo combining multiple components."""

from rich.console import Console
from rich.panel import Panel
from rich.table import Table
from rich.tree import Tree
from rich.syntax import Syntax
from rich.columns import Columns
from rich.markdown import Markdown
from rich.text import Text
from rich import box

console = Console()

# Title
console.rule("[bold bright_magenta]Rich Components Showcase")
console.print()

# 1. Panel with markdown
md_content = Markdown("""\
## Features
- **Tables** with custom borders
- *Syntax highlighting* for code
- Tree views for hierarchical data
""")
console.print(Panel(md_content, title="About This Demo", border_style="cyan"))
console.print()

# 2. Syntax highlighting
code = '''\
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await db.fetch_user(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return {"user": user}
'''
syntax = Syntax(code, "python", theme="monokai", line_numbers=True, padding=1)
console.print(Panel(syntax, title="API Endpoint", border_style="green"))
console.print()

# 3. Tree
tree = Tree("[bold]Project Structure[/bold]")
src = tree.add(":open_file_folder: src")
src.add(":page_facing_up: main.py")
src.add(":page_facing_up: config.py")
models = src.add(":open_file_folder: models")
models.add(":page_facing_up: user.py")
models.add(":page_facing_up: post.py")
tests = tree.add(":open_file_folder: tests")
tests.add(":page_facing_up: test_main.py")
tree.add(":page_facing_up: pyproject.toml")

console.print(Panel(tree, title="Directory Layout", border_style="yellow"))
console.print()

# 4. Table
table = Table(title="API Endpoints", box=box.ROUNDED, border_style="blue")
table.add_column("Method", style="bold")
table.add_column("Path", style="cyan")
table.add_column("Description")
table.add_column("Auth", justify="center")

table.add_row("[green]GET[/green]", "/users", "List all users", ":unlocked:")
table.add_row("[green]GET[/green]", "/users/{id}", "Get user by ID", ":unlocked:")
table.add_row("[yellow]POST[/yellow]", "/users", "Create user", ":locked:")
table.add_row("[blue]PUT[/blue]", "/users/{id}", "Update user", ":locked:")
table.add_row("[red]DELETE[/red]", "/users/{id}", "Delete user", ":locked:")

console.print(table)
console.print()

# 5. Columns of panels
panels = [
    Panel("[bold green]200[/bold green]\nOK", title="Success", border_style="green", expand=True),
    Panel("[bold yellow]301[/bold yellow]\nRedirect", title="Redirect", border_style="yellow", expand=True),
    Panel("[bold red]404[/bold red]\nNot Found", title="Client Error", border_style="red", expand=True),
    Panel("[bold bright_red]500[/bold bright_red]\nServer Error", title="Server Error", border_style="bright_red", expand=True),
]
console.print(Columns(panels, equal=True, expand=True))

console.print()
console.rule("[bold bright_magenta]End of Showcase")
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
