---
name: jinja2
description: Fast template engine with inheritance, macros, and sandboxed execution. Use when rendering templates, generating HTML/config/code from templates, using template inheritance, or building DSLs with template syntax. Triggers on jinja2, jinja, template engine, template rendering, template inheritance, code generation templates.
---

# Jinja2 — Template Engine (v3.1.5)

## Quick Start

```bash
pip install Jinja2
```

```python
from jinja2 import Template

template = Template("Hello, {{ name }}!")
template.render(name="World")  # "Hello, World!"
```

## Key Patterns

### Environment with file loader
```python
from jinja2 import Environment, FileSystemLoader

env = Environment(
    loader=FileSystemLoader("templates/"),
    autoescape=True,
    trim_blocks=True,
    lstrip_blocks=True,
)
template = env.get_template("page.html")
output = template.render(title="Home", items=["a", "b"])
```

### Template syntax essentials
```jinja
{{ user.name }}                          {# variables #}
{{ items | join(", ") }}                 {# filters #}
{% for item in items %}{{ item }}{% endfor %}
{% if user.admin %}Admin{% else %}User{% endif %}
```

### Template inheritance
```jinja
{# base.html #}
<html><body>{% block content %}{% endblock %}</body></html>

{# page.html #}
{% extends "base.html" %}
{% block content %}<h1>{{ title }}</h1>{% endblock %}
```

## References

- **[api.md](references/api.md)** — Environment, loaders, template syntax (filters, tests, control structures, inheritance, macros), autoescaping, custom filters, extensions, sandboxing, and async support
- **[examples.md](references/examples.md)** — Gotchas (autoescape, scoping, whitespace) and complete examples (HTML emails, config generation, sandboxed templates)
