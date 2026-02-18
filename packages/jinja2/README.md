# Jinja2

A fast, expressive, and extensible **template engine** for Python.

`Jinja2` is the most widely used Python templating library. It powers Flask's template rendering, Ansible playbooks, Cookiecutter project templates, Salt configuration, and many other tools. Jinja2 provides a rich template language with variables, filters, tests, control structures, template inheritance, macros, and sandboxed execution -- all with a clean syntax inspired by Django's template language but with full expression support.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `Jinja2` |
| **Latest version** | 3.1.5 (released 2025) |
| **Python support** | Python >= 3.8 |
| **License** | BSD 3-Clause |
| **Repository** | https://github.com/pallets/jinja |
| **Documentation** | https://jinja.palletsprojects.com |

The import name is `jinja2`.

---

## Installation

```bash
pip install Jinja2
```

Optional dependency for autoescaping in HTML templates:

```bash
pip install MarkupSafe   # installed automatically with Jinja2
```

```python
import jinja2
from jinja2 import Environment, FileSystemLoader, Template
```

---

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

---

## Gotchas and Common Mistakes

### 1. Autoescaping Is Off by Default

Unlike Django templates, Jinja2 does **not** autoescape by default. For HTML templates, always enable it:

```python
env = Environment(autoescape=True)
```

### 2. `StrictUndefined` for Catching Typos

By default, undefined variables silently render as empty strings. Use `StrictUndefined` during development:

```python
from jinja2 import Environment, StrictUndefined

env = Environment(undefined=StrictUndefined)
# Now {{ typo_variable }} raises UndefinedError instead of rendering empty
```

### 3. Template Variables Are Not Python Scope

Variables set in a `for` loop or `if` block do not leak out to the enclosing scope (unlike Python). Use `{% set %}` in the outer scope or the namespace object:

```jinja
{# This does NOT work as expected #}
{% set found = false %}
{% for item in items %}
  {% if item.match %}
    {% set found = true %}  {# sets a NEW variable in loop scope #}
  {% endif %}
{% endfor %}
{{ found }}  {# still false! #}

{# Use namespace instead #}
{% set ns = namespace(found=false) %}
{% for item in items %}
  {% if item.match %}
    {% set ns.found = true %}
  {% endif %}
{% endfor %}
{{ ns.found }}  {# correctly true #}
```

### 4. Whitespace Control

Block tags generate whitespace. Use `-` to strip it:

```jinja
{% for item in items -%}
  {{ item }}
{%- endfor %}
```

Or use `trim_blocks=True` and `lstrip_blocks=True` in the Environment.

### 5. Calling `render()` vs `generate()`

`render()` returns the entire output as a string. `generate()` returns a generator for streaming large templates:

```python
for chunk in template.generate(data=large_dataset):
    response.write(chunk)
```

---

## Complete Examples

### Example 1: HTML Email Template

```python
from jinja2 import Environment, FileSystemLoader, select_autoescape

env = Environment(
    loader=FileSystemLoader("templates/"),
    autoescape=select_autoescape(["html"]),
)

template = env.from_string("""
<html>
<body>
  <h1>Hello, {{ user.name }}!</h1>
  <p>Your order #{{ order.id }} contains:</p>
  <ul>
  {% for item in order.items %}
    <li>{{ item.name }} - {{ item.price|currency }}</li>
  {% endfor %}
  </ul>
  <p><strong>Total: {{ order.total|currency }}</strong></p>
</body>
</html>
""")

def currency(value):
    return f"${value:,.2f}"

env.filters["currency"] = currency

html = template.render(
    user={"name": "Alice"},
    order={"id": 1234, "items": [
        {"name": "Widget", "price": 9.99},
        {"name": "Gadget", "price": 24.99},
    ], "total": 34.98},
)
```

### Example 2: Configuration File Generation

```python
from jinja2 import Environment

env = Environment()
template = env.from_string("""
server {
    listen {{ port }};
    server_name {{ domain }};

    {% for location in locations %}
    location {{ location.path }} {
        proxy_pass {{ location.backend }};
    }
    {% endfor %}
}
""")

config = template.render(
    port=443,
    domain="example.com",
    locations=[
        {"path": "/api", "backend": "http://127.0.0.1:8000"},
        {"path": "/static", "backend": "http://127.0.0.1:9000"},
    ],
)
print(config)
```

### Example 3: Sandboxed User Templates

```python
from jinja2.sandbox import SandboxedEnvironment

env = SandboxedEnvironment()

# Safe user-provided template
user_template = "Hello {{ name }}, you have {{ count }} messages."
template = env.from_string(user_template)
print(template.render(name="Bob", count=5))

# Malicious template -- blocked
try:
    evil = env.from_string("{{ ''.__class__.__mro__[1].__subclasses__() }}")
    evil.render()
except Exception as e:
    print(f"Blocked: {e}")
```

---

## See Also

- [Jinja2 Documentation](https://jinja.palletsprojects.com)
- [Jinja2 on GitHub](https://github.com/pallets/jinja)
- [Template Designer Documentation](https://jinja.palletsprojects.com/en/3.1.x/templates/)
- [Jinja2 Extensions API](https://jinja.palletsprojects.com/en/3.1.x/extensions/)
