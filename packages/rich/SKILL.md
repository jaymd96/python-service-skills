---
name: rich
description: Rich text and beautiful formatting for the terminal. Use when adding colored output, tables, progress bars, syntax highlighting, pretty tracebacks, or live displays to CLI applications. Triggers on rich, terminal formatting, progress bars, console tables, pretty print, colored output, rich traceback.
---

# rich — Terminal Formatting (v13.9.4)

## Quick Start

```bash
pip install rich
```

```python
from rich.console import Console

console = Console()
console.print("[bold red]Error:[/] Something went wrong")
console.print({"key": "value"})  # auto pretty-prints
```

## Key Patterns

### Tables
```python
from rich.table import Table

table = Table(title="Users")
table.add_column("Name", style="cyan")
table.add_column("Email")
table.add_row("Alice", "alice@ex.com")
console.print(table)
```

### Progress bars
```python
from rich.progress import track

for item in track(items, description="Processing..."):
    process(item)
```

### Pretty tracebacks (install globally)
```python
from rich.traceback import install
install(show_locals=True)  # replaces default traceback handler
```

## References

- **[api.md](references/api.md)** — Console class, markup/styles, themes, text formatting
- **[components.md](references/components.md)** — Table, Panel, Tree, Progress, Status, Layout, Syntax
- **[display.md](references/display.md)** — Logging handler, tracebacks, inspect, export (HTML/SVG), prompts
- **[examples.md](references/examples.md)** — Complete examples, gotchas, CI/threading tips

## Grep Patterns

- `from rich` — Find rich imports
- `Console\(` — Find console instances
- `rich\.print|console\.print` — Find rich print usage
- `Table\(|Progress\(` — Find component usage
