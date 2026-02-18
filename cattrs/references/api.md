# cattrs — API Reference

> Part of the cattrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

1. [cattrs.structure()](#cattrsstructuredata-type----structuring-dict---object)
2. [cattrs.unstructure()](#cattrsunstructureobj----unstructuring-object---dict)
3. [cattrs.Converter()](#cattrsconverter----the-main-converter-class)
4. [cattrs.structure_attrs_from_dict](#cattrsstructure_attrs_from_dict----default-attrs-structuring)
5. [cattrs.structure_attrs_from_tuple](#cattrsstructure_attrs_from_tuple----tuple-based-structuring)
6. [API Quick Reference](#api-quick-reference)

---

### `cattrs.structure(data, type)` -- Structuring (dict -> object)

The primary function. Takes unstructured data and a target type, returns a typed object.

```python
import cattrs
import attr

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

raw = {"name": "Alice", "age": 30}
user = cattrs.structure(raw, User)
# User(name='Alice', age=30)

assert isinstance(user, User)
assert user.name == "Alice"
```

`cattrs.structure` is a convenience function that calls `structure` on the global `Converter` instance. It works with dicts, lists, primitives, and nested structures.

```python
# Structuring a list of objects
raw_users = [
    {"name": "Alice", "age": 30},
    {"name": "Bob", "age": 25},
]
users = cattrs.structure(raw_users, list[User])
# [User(name='Alice', age=30), User(name='Bob', age=25)]
```

### `cattrs.unstructure(obj)` -- Unstructuring (object -> dict)

The inverse operation. Takes a typed object and converts it to plain dicts/lists/primitives.

```python
import cattrs
import attr

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

user = User(name="Alice", age=30)
raw = cattrs.unstructure(user)
# {'name': 'Alice', 'age': 30}

assert isinstance(raw, dict)
assert raw == {"name": "Alice", "age": 30}
```

Unstructuring works recursively. Nested attrs/dataclass objects are converted to nested dicts:

```python
import attr
import cattrs
from typing import List

@attr.s(auto_attribs=True)
class Address:
    street: str
    city: str

@attr.s(auto_attribs=True)
class User:
    name: str
    addresses: List[Address]

user = User(
    name="Alice",
    addresses=[
        Address(street="123 Main St", city="Springfield"),
        Address(street="456 Oak Ave", city="Shelbyville"),
    ],
)

raw = cattrs.unstructure(user)
# {
#     'name': 'Alice',
#     'addresses': [
#         {'street': '123 Main St', 'city': 'Springfield'},
#         {'street': '456 Oak Ave', 'city': 'Shelbyville'},
#     ],
# }
```

### `cattrs.Converter()` -- The Main Converter Class

The `Converter` is the central object in cattrs. It holds all the structuring/unstructuring hooks and configuration. The module-level `cattrs.structure` and `cattrs.unstructure` functions use a default global converter, but in real applications you should create your own.

```python
import cattrs

# Create your own converter
converter = cattrs.Converter()

# Use it for structuring and unstructuring
user = converter.structure({"name": "Alice", "age": 30}, User)
raw = converter.unstructure(user)
```

**Why create your own converter:**

- **Isolation**: Different parts of your app can have different serialization rules (e.g., API layer vs. database layer).
- **Custom hooks**: You register type-specific hooks on a specific converter.
- **Thread safety**: Each converter is independent. No shared mutable global state.
- **Testing**: Easy to create test converters with different behavior.

```python
# API converter -- uses camelCase keys
api_converter = cattrs.Converter()

# Database converter -- uses snake_case keys
db_converter = cattrs.Converter()

# Each can have completely different hooks registered
```

### `cattrs.structure_attrs_from_dict` -- Default attrs Structuring

The default structuring function used for attrs classes. Takes a dict and produces an attrs instance by matching dict keys to attrs field names.

```python
import cattrs
import attr

@attr.s(auto_attribs=True)
class Point:
    x: float
    y: float

# This is what cattrs uses internally for attrs classes:
point = cattrs.structure_attrs_from_dict({"x": 1.0, "y": 2.0}, Point)
# Point(x=1.0, y=2.0)
```

You rarely call this directly, but it is useful when writing custom hooks that need to fall back to default behavior:

```python
def custom_point_hook(data, type_):
    # Add custom preprocessing...
    data = {k.lower(): v for k, v in data.items()}
    # Then delegate to the default
    return cattrs.structure_attrs_from_dict(data, type_)

converter.register_structure_hook(Point, custom_point_hook)
```

### `cattrs.structure_attrs_from_tuple` -- Tuple-Based Structuring

Structure an attrs class from a tuple (positional) instead of a dict (named).

```python
import cattrs
import attr

@attr.s(auto_attribs=True)
class Point:
    x: float
    y: float

# Structure from positional tuple
point = cattrs.structure_attrs_from_tuple((1.0, 2.0), Point)
# Point(x=1.0, y=2.0)
```

Useful when working with compact data formats (CSV rows, database result tuples, msgpack arrays).

---

## API Quick Reference

```
# cattrs API Reference
# Version: 24.1.2 | Python: 3.8+ | Dependency: attrs >= 20.1.0

## Module-Level Functions (use the global converter)

cattrs.structure(data, type)                    # Dict -> typed object
cattrs.unstructure(obj)                         # Typed object -> dict
cattrs.structure_attrs_from_dict(data, type)    # Default attrs structuring
cattrs.structure_attrs_from_tuple(data, type)   # Tuple-based attrs structuring

## Converter Class

c = cattrs.Converter()                          # Create a converter
c = cattrs.GenConverter()                       # Create a code-generating converter

c.structure(data, type)                         # Structure data into type
c.unstructure(obj)                              # Unstructure obj to primitives

c.register_structure_hook(type, func)           # Custom structuring for a type
c.register_unstructure_hook(type, func)         # Custom unstructuring for a type

c.register_structure_hook_func(predicate, func)   # Predicate-based structuring hook
c.register_unstructure_hook_func(predicate, func) # Predicate-based unstructuring hook

## Code Generation (cattrs.gen)

from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override

make_dict_structure_fn(cls, converter, **field_overrides)
make_dict_unstructure_fn(cls, converter, **field_overrides)

override(
    rename="new_key",           # Rename field key in dict
    omit=True,                  # Skip this field
    omit_if_default=True,       # Skip if value equals default
    struct_hook=callable,       # Custom structuring for this field
    unstruct_hook=callable,     # Custom unstructuring for this field
)

## Preconf Converters

from cattrs.preconf.json import make_converter      # JSON (stdlib)
from cattrs.preconf.orjson import make_converter     # orjson
from cattrs.preconf.msgpack import make_converter    # msgpack
from cattrs.preconf.bson import make_converter       # BSON
from cattrs.preconf.tomlkit import make_converter    # TOML
from cattrs.preconf.pyyaml import make_converter     # YAML

## Strategies

from cattrs.strategies import include_subclasses       # Polymorphic structuring
from cattrs.strategies import configure_tagged_union   # Discriminated unions
from cattrs.strategies import use_class_methods        # Delegate to class methods

## Errors

cattrs.errors.ClassValidationError     # Detailed per-field validation errors
```
