# Jinja2 — Examples & Gotchas

> Part of the jinja2 skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Autoescaping Is Off by Default](#1-autoescaping-is-off-by-default)
  - [2. StrictUndefined for Catching Typos](#2-strictundefined-for-catching-typos)
  - [3. Template Variables Are Not Python Scope](#3-template-variables-are-not-python-scope)
  - [4. Whitespace Control](#4-whitespace-control)
  - [5. Calling render() vs generate()](#5-calling-render-vs-generate)
- [Complete Examples](#complete-examples)
  - [Example 1: HTML Email Template](#example-1-html-email-template)
  - [Example 2: Configuration File Generation](#example-2-configuration-file-generation)
  - [Example 3: Sandboxed User Templates](#example-3-sandboxed-user-templates)
- [See Also](#see-also)

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
