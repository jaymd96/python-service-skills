---
name: rune
description: Composable validation and serialization built on attrs/cattrs. Use when defining models with @inscribed, parsing with .parse(), adding Annotated constraints (Gt, MinLen, Pattern), field/model validators, Converter serialization, camelCase aliases, TypeRegistry, JSON Schema generation, or handling ValidationError. Triggers on rune, inscribed, validation, constraints, Annotated, Converter, json_schema, TypeRegistry.
---

# Rune — Composable Validation & Serialization (v0.1.0)

## Quick Start

```bash
pip install rune
```

```python
import attrs
from typing import Annotated
from rune import inscribed, Gt, MinLen, Pattern, Converter

@inscribed
@attrs.define
class User:
    name: Annotated[str, MinLen(1)]
    age: Annotated[int, Gt(0)]
    code: Annotated[str, Pattern(r'^[a-z][a-z0-9-]*$')]
    email: str | None = None

# Validated construction from dict
user = User.parse({"name": "Alice", "age": 30, "code": "alice-1"})

# Fast construction (no validation)
user = User(name="Alice", age=30, code="alice-1")

# Serialization
c = Converter()
d = c.unstructure(user, mode="json", by_alias=True)
```

## Key Patterns

### Constraints and validation
```python
from typing import Annotated
from rune import Gt, Ge, Lt, Le, MinLen, MaxLen, Pattern, MinItems, UniqueItems
from rune import field_validator, model_validator

age: Annotated[int, Gt(0), Lt(150)]
name: Annotated[str, MinLen(1), MaxLen(63)]
tags: Annotated[list[str], MinItems(1), UniqueItems()]

@field_validator("version")
@classmethod
def validate_version(cls, v): ...

@model_validator
@classmethod
def validate_dates(cls, instance): ...
```

### Converter with modes and aliases
```python
from rune import Converter, camel_case_aliases, computed

c = Converter(mode="json", by_alias=True)
d = c.unstructure(user)           # JSON-safe, aliased keys
user = c.structure(data, User)    # dict -> validated instance
```

## References

- **[overview.md](references/overview.md)** — Full Rune framework overview, API quick reference, Apollo integration examples, best practices
- **[api.md](references/api.md)** — @inscribed, field(), constraints, validators, Converter, TypeRegistry, CoercionRegistry
- **[serialization.md](references/serialization.md)** — Converter modes, aliases, computed fields, discriminated unions, JSON Schema
- **[examples.md](references/examples.md)** — Complete examples, integration patterns, gotchas

## Grep Patterns

- `@inscribed|\.parse\(` — Find Rune model definitions and parsing
- `Annotated\[.*(?:Gt|MinLen|Pattern)` — Find constraint usage
- `Converter\(|field_validator|model_validator` — Find serialization and validation
