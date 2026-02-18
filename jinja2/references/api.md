# Jinja2 — API Reference

> Part of the jinja2 skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API Reference](#core-api-reference)
  - [Template -- Quick One-Off Rendering](#template----quick-one-off-rendering)
  - [Environment -- Full Configuration](#environment----full-configuration)
  - [Template Loaders](#template-loaders)
- [Template Syntax](#template-syntax)
  - [Variables](#variables)
  - [Filters](#filters)
  - [Tests](#tests)
  - [Control Structures](#control-structures)
  - [Template Inheritance](#template-inheritance)
  - [Includes](#includes)
  - [Macros](#macros)
- [Autoescaping](#autoescaping)
- [Custom Filters](#custom-filters)
- [Custom Extensions](#custom-extensions)
- [Sandboxed Execution](#sandboxed-execution)
- [Async Support](#async-support)

## Core API Reference

### `Template` -- Quick One-Off Rendering

```python
from jinja2 import Template

template = Template("Hello, {{ name }}!")
result = template.render(name="World")
print(result)  # "Hello, World!"
```

### `Environment` -- Full Configuration

The `Environment` is the central configuration object. It stores shared settings, filters, globals, and the template loader.

```python
from jinja2 import Environment, FileSystemLoader

env = Environment(
    loader=FileSystemLoader("templates/"),
    autoescape=True,          # HTML-escape by default
    trim_blocks=True,         # strip first newline after block tags
    lstrip_blocks=True,       # strip leading whitespace before block tags
)

template = env.get_template("index.html")
output = template.render(title="Home", items=["a", "b", "c"])
```

#### Key `Environment` Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `loader` | `None` | Where to find templates (FileSystemLoader, PackageLoader, etc.) |
| `autoescape` | `False` | Auto-escape HTML. Set to `True` or a callable |
| `trim_blocks` | `False` | Remove first newline after a block tag |
| `lstrip_blocks` | `False` | Strip leading whitespace (including tabs) from block tag lines |
| `undefined` | `Undefined` | Behavior for undefined variables (`Undefined`, `StrictUndefined`, `DebugUndefined`) |
| `extensions` | `[]` | List of Jinja2 extension classes to load |

### Template Loaders

| Loader | Description |
|--------|-------------|
| `FileSystemLoader(path)` | Load from filesystem directories |
| `PackageLoader(package, path)` | Load from Python package data |
| `DictLoader(mapping)` | Load from a dict of `{name: source}` |
| `FunctionLoader(func)` | Load via a callable |
| `ChoiceLoader([loaders])` | Try multiple loaders in order |
| `PrefixLoader({"prefix": loader})` | Route by prefix |

---

## Template Syntax

### Variables

```jinja
{{ variable }}
{{ user.name }}
{{ items[0] }}
{{ data["key"] }}
```

### Filters

Filters transform values. They are applied with the pipe `|` operator:

```jinja
{{ name|upper }}
{{ items|join(", ") }}
{{ text|truncate(80) }}
{{ value|default("N/A") }}
{{ html_content|safe }}         {# mark as safe, skip autoescaping #}
{{ list|sort|first }}
```

#### Common Built-in Filters

| Filter | Description |
|--------|-------------|
| `upper`, `lower`, `title`, `capitalize` | Case conversion |
| `trim`, `striptags` | Whitespace / HTML tag removal |
| `replace(old, new)` | String replacement |
| `truncate(length)` | Truncate with ellipsis |
| `default(value)` | Default for undefined/falsy |
| `join(separator)` | Join a list into a string |
| `sort`, `reverse`, `unique` | List operations |
| `first`, `last`, `length` | List accessors |
| `int`, `float`, `string` | Type conversion |
| `batch(n)`, `slice(n)` | Split into groups |
| `tojson` | JSON-encode |
| `safe` | Mark as safe (no escaping) |
| `escape` / `e` | Force HTML escaping |
| `indent(width)` | Indent multiline strings |
| `map(attribute=...)` | Map over list items |
| `select(test)` / `reject(test)` | Filter list items |
| `groupby(attribute)` | Group items by attribute |

### Tests

Tests check conditions. Used with `is`:

```jinja
{% if value is defined %}...{% endif %}
{% if number is odd %}...{% endif %}
{% if name is string %}...{% endif %}
{% if items is iterable %}...{% endif %}
```

### Control Structures

```jinja
{# For loops #}
{% for item in items %}
  {{ item }}
{% else %}
  No items found.
{% endfor %}

{# Loop special variables #}
{% for item in items %}
  {{ loop.index }}     {# 1-based index #}
  {{ loop.index0 }}    {# 0-based index #}
  {{ loop.first }}     {# True on first iteration #}
  {{ loop.last }}      {# True on last iteration #}
  {{ loop.length }}    {# Total number of items #}
{% endfor %}

{# Conditionals #}
{% if user.is_admin %}
  Admin panel
{% elif user.is_staff %}
  Staff panel
{% else %}
  User panel
{% endif %}

{# Set variables #}
{% set greeting = "Hello" %}
```

### Template Inheritance

Base template (`base.html`):

```jinja
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}Default Title{% endblock %}</title>
</head>
<body>
  {% block content %}{% endblock %}
  {% block footer %}
    <footer>Default footer</footer>
  {% endblock %}
</body>
</html>
```

Child template (`page.html`):

```jinja
{% extends "base.html" %}

{% block title %}My Page{% endblock %}

{% block content %}
  <h1>Welcome</h1>
  <p>This overrides the content block.</p>
{% endblock %}

{# footer block is inherited as-is from base.html #}
```

### Includes

```jinja
{% include "header.html" %}
{% include "sidebar.html" ignore missing %}
{% include ["special.html", "default.html"] %}  {# first found #}
```

### Macros

Macros are reusable template functions:

```jinja
{% macro input(name, type="text", value="") %}
  <input type="{{ type }}" name="{{ name }}" value="{{ value }}">
{% endmacro %}

{{ input("username") }}
{{ input("password", type="password") }}
```

Import macros from another template:

```jinja
{% from "forms.html" import input, textarea %}
{{ input("email") }}
```

---

## Autoescaping

Autoescaping prevents XSS vulnerabilities by HTML-encoding output:

```python
env = Environment(autoescape=True)
# or: autoescape=select_autoescape(["html", "xml"])

from jinja2 import select_autoescape
env = Environment(
    loader=FileSystemLoader("templates/"),
    autoescape=select_autoescape(["html", "htm", "xml"]),
)
```

In templates, use `|safe` to mark trusted content that should not be escaped:

```jinja
{{ user_input }}          {# auto-escaped: &lt;script&gt; #}
{{ trusted_html|safe }}   {# NOT escaped #}
```

---

## Custom Filters

```python
from jinja2 import Environment

def currency(value, symbol="$"):
    return f"{symbol}{value:,.2f}"

env = Environment()
env.filters["currency"] = currency

template = env.from_string("Price: {{ amount|currency }}")
template.render(amount=1234.5)  # "Price: $1,234.50"

template = env.from_string("Price: {{ amount|currency('EUR ') }}")
template.render(amount=1234.5)  # "Price: EUR 1,234.50"
```

---

## Custom Extensions

Extensions add tags or behavior to the template engine:

```python
from jinja2 import Environment
from jinja2.ext import Extension

# Built-in extensions
env = Environment(extensions=[
    "jinja2.ext.i18n",       # Internationalization
    "jinja2.ext.loopcontrols",  # break / continue in loops
    "jinja2.ext.debug",      # {% debug %} tag
])
```

With `loopcontrols`:

```jinja
{% for item in items %}
  {% if item.hidden %}{% continue %}{% endif %}
  {% if loop.index > 10 %}{% break %}{% endif %}
  {{ item.name }}
{% endfor %}
```

---

## Sandboxed Execution

`SandboxedEnvironment` restricts what templates can access, suitable for untrusted templates:

```python
from jinja2.sandbox import SandboxedEnvironment

env = SandboxedEnvironment()

# This prevents access to dangerous attributes and methods
template = env.from_string("{{ items.__class__.__mro__ }}")
# Raises SecurityError
```

---

## Async Support

Jinja2 supports async rendering for use with `asyncio`:

```python
from jinja2 import Environment

env = Environment(enable_async=True)
template = env.from_string("Hello, {{ name }}!")

# In an async context:
import asyncio

async def render():
    result = await template.render_async(name="World")
    print(result)

asyncio.run(render())
```

Async is useful when template variables are coroutines or async generators.
