# Rich — Examples & Gotchas

> Part of the rich skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Console Width Detection in CI/Containers](#1-console-width-detection-in-cicontainers)
  - [2. Capturing Output (StringIO vs Console)](#2-capturing-output-stringio-vs-console)
  - [3. Color Support Detection](#3-color-support-detection)
  - [4. Performance with Very Large Outputs](#4-performance-with-very-large-outputs)
  - [5. Thread Safety with Console](#5-thread-safety-with-console)
  - [6. Markup in f-strings with User Data](#6-markup-in-f-strings-with-user-data)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Styled Console Output](#example-1-styled-console-output)
  - [Example 2: Building Tables](#example-2-building-tables)
  - [Example 3: Progress Bars for Downloads](#example-3-progress-bars-for-downloads)
  - [Example 4: Live Updating Display](#example-4-live-updating-display)
  - [Example 5: Custom Themes](#example-5-custom-themes)
  - [Example 6: Logging with RichHandler](#example-6-logging-with-richhandler)
  - [Example 7: Pretty Tracebacks](#example-7-pretty-tracebacks)
  - [Example 8: Rich Inspect](#example-8-rich-inspect)
  - [Example 9: Combining Components (Full Application)](#example-9-combining-components-full-application)
- [Further Reading](#further-reading)

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

### 2. Capturing Output (StringIO vs Console)

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

## Further Reading

- **Official Documentation:** [https://rich.readthedocs.io/en/stable/](https://rich.readthedocs.io/en/stable/)
- **GitHub Repository:** [https://github.com/Textualize/rich](https://github.com/Textualize/rich)
- **PyPI Page:** [https://pypi.org/project/rich/](https://pypi.org/project/rich/)
- **Textual** (TUI framework built on Rich): [https://github.com/Textualize/textual](https://github.com/Textualize/textual)
- **Rich CLI** (command-line tool): [https://github.com/Textualize/rich-cli](https://github.com/Textualize/rich-cli)
