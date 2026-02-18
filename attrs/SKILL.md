---
name: attrs
description: Python classes without boilerplate using attrs. Use when defining data classes with @attrs.define or @attrs.frozen, adding validators or converters, creating immutable value objects, or working with the attrs/cattrs ecosystem. Triggers on requests involving attrs, define, frozen, field validators, evolve, attrs classes, data classes with validation.
---

# attrs — Python classes without boilerplate (v24.3.0)

## Quick Start

```bash
pip install attrs
```

```python
import attrs

@attrs.define
class User:
    name: str
    email: str
    age: int = 0

@attrs.frozen  # immutable + hashable
class Point:
    x: float
    y: float

u = User("Alice", "alice@ex.com", 30)
p = attrs.evolve(Point(1.0, 2.0), x=3.0)  # Point(x=3.0, y=2.0)
```

## Key Patterns

### Modern API (always use this)
```python
@attrs.define      # mutable, slots=True by default
@attrs.frozen      # immutable + hashable
attrs.field()      # field config: validator, converter, factory, alias
attrs.Factory(list)  # mutable defaults (NOT = [] !)
```

### Validators, converters, and evolve
```python
@attrs.define
class Config:
    port: int = attrs.field(converter=int, validator=attrs.validators.ge(1))
    tags: list[str] = attrs.Factory(list)

@attrs.frozen
class Settings:
    debug: bool = False
dev = attrs.evolve(Settings(), debug=True)
```

## References

- **[api.md](references/api.md)** — @define, field(), frozen, slots, Factory, make_class
- **[validators.md](references/validators.md)** — Built-in validators, custom validators, converters
- **[advanced.md](references/advanced.md)** — evolve, asdict/astuple, comparison, hashing, aliases, on_setattr
- **[examples.md](references/examples.md)** — Complete examples, gotchas, migration patterns

## Grep Patterns

- `@define|@attrs` — Find attrs class definitions
- `field\(|attrib\(` — Find field declarations
- `validator|converter` — Find validation/conversion logic
