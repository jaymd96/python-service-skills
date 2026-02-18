# Rich — Components

> Part of the rich skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Table -- Configurable Tables](#table----configurable-tables)
- [Panel -- Boxed Content](#panel----boxed-content)
- [Columns -- Multi-Column Layout](#columns----multi-column-layout)
- [Tree -- Tree Display](#tree----tree-display)
- [Syntax -- Syntax-Highlighted Code](#syntax----syntax-highlighted-code)
- [Markdown -- Rendered Markdown](#markdown----rendered-markdown)
- [Progress -- Progress Bars](#progress----progress-bars)
- [Prompt / Confirm -- User Input](#prompt--confirm----user-input)
- [Pretty -- Auto-Formatting Data Structures](#pretty----auto-formatting-data-structures)
- [Layout -- Split-Screen Terminal Layouts](#layout----split-screen-terminal-layouts)
- [Live -- Live-Updating Displays](#live----live-updating-displays)

---

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
