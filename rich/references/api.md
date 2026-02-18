# Rich — API Reference

> Part of the rich skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Console() -- The Main Interface](#console----the-main-interface)
- [console.print() -- Rich Printing](#consoleprint----rich-printing)
- [console.log() -- Logging with Timestamps](#consolelog----logging-with-timestamps)
- [console.rule() -- Horizontal Rule](#consolerule----horizontal-rule)
- [console.status() -- Spinner Context Manager](#consolestatus----spinner-context-manager)
- [Rich Markup](#rich-markup)
- [Styling System](#styling-system)
  - [Style Strings](#style-strings)
  - [Style Objects](#style-objects)
  - [Theme -- Consistent Styling](#theme----consistent-styling)
  - [Console Markup Syntax](#console-markup-syntax)

---

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
