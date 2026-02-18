# cattrs -- Composable Converters for Structured Data in Python

> Complex custom class hierarchies? cattrs structures and unstructures them with speed and precision.

**cattrs** is a Python library for composable conversion between unstructured data (dicts, lists, primitives) and structured data (attrs classes, dataclasses, typed objects). It is the serialization/deserialization backbone of the witchcraft stack (Rune, Transmute, Enchant), handling all transformations between raw wire formats and typed Python objects.

| Detail | Value |
|---|---|
| **PyPI** | [cattrs](https://pypi.org/project/cattrs/) |
| **Latest stable version** | **24.1.2** (verify at PyPI for newer releases) |
| **Python** | 3.8+ |
| **Source** | [github.com/python-attrs/cattrs](https://github.com/python-attrs/cattrs) |
| **Docs** | [catt.rs](https://catt.rs/en/stable/) |
| **License** | MIT |
| **Dependencies** | `attrs >= 20.1.0` |

**Core Concepts:**

- **Structuring** (dict -> object): Take unstructured data (dicts, lists, primitives from JSON/API responses/databases) and convert it into fully typed Python objects.
- **Unstructuring** (object -> dict): Take typed Python objects and convert them back to plain dicts/lists suitable for serialization to JSON, msgpack, BSON, etc.
- **Converters**: Stateful, configurable, composable objects that hold the rules for how types are structured and unstructured.
- **Hooks**: Per-type functions registered on a converter that control exactly how a specific type is handled.

**Why cattrs exists:** Libraries like `json.loads()` give you dicts and lists. You want typed objects. Libraries like `attrs` and `dataclasses` give you typed objects, but no serialization. cattrs bridges the gap -- it converts between the two worlds with full type awareness, composability, and speed.

---

## Table of Contents

1. [Installation](#installation)
2. [Core API](#core-api)
3. [Preconf Converters](#preconf-converters)
4. [Strategy System](#strategy-system)
5. [GenConverter](#genconverter)
6. [Type Support](#type-support)
7. [Detailed Hooks](#detailed-hooks)
8. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
9. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
pip install cattrs
```

Or with specific format support (extras install the format library plus cattrs):

```bash
pip install cattrs[orjson]    # For orjson preconf converter
pip install cattrs[msgpack]   # For msgpack preconf converter
pip install cattrs[bson]      # For BSON preconf converter
pip install cattrs[tomlkit]   # For TOML preconf converter
pip install cattrs[pyyaml]    # For YAML preconf converter
```

In `pyproject.toml`:

```toml
[project]
dependencies = [
    "cattrs>=24.1.0",
]
```

In `requirements.txt`:

```
cattrs>=24.1.0
```

---

## Core API

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

### `cattrs.GenConverter()` / `cattrs.GEN_CONVERTER` -- Code-Generating Converter

`GenConverter` (also available as the class `cattrs.GenConverter`) is a subclass of `Converter` that generates specialized structuring/unstructuring code at hook registration time rather than using generic dispatch at runtime. This makes it significantly faster.

```python
import cattrs

# Create a code-generating converter
converter = cattrs.GenConverter()

# Use exactly like a regular Converter
user = converter.structure({"name": "Alice", "age": 30}, User)
raw = converter.unstructure(user)
```

`cattrs.GEN_CONVERTER` was historically a pre-built singleton `GenConverter` instance. In modern cattrs, prefer creating your own `GenConverter()`.

> **Note**: As of cattrs 24.x, the default `Converter` itself uses generated code for attrs classes and dataclasses, so the performance gap has narrowed. `GenConverter` still offers additional features like `omit` and `rename` overrides. See the [GenConverter section](#genconverter) for details.

### `converter.register_structure_hook(type, func)` -- Custom Structuring

Register a custom function to control how a specific type is structured (deserialized).

```python
import cattrs
from datetime import datetime

converter = cattrs.Converter()

# Register a hook for datetime: expect ISO format strings
def structure_datetime(val, _):
    return datetime.fromisoformat(val)

converter.register_structure_hook(datetime, structure_datetime)

# Now the converter knows how to structure datetimes
dt = converter.structure("2024-01-15T10:30:00", datetime)
# datetime(2024, 1, 15, 10, 30)
```

The hook function signature is `func(value, type) -> structured_value`:

- `value`: The raw unstructured data.
- `type`: The target type (useful for generic hooks).
- Returns: The structured (typed) object.

### `converter.register_unstructure_hook(type, func)` -- Custom Unstructuring

Register a custom function to control how a specific type is unstructured (serialized).

```python
import cattrs
from datetime import datetime

converter = cattrs.Converter()

# Unstructure datetimes to ISO strings
def unstructure_datetime(val):
    return val.isoformat()

converter.register_unstructure_hook(datetime, unstructure_datetime)

dt = datetime(2024, 1, 15, 10, 30)
result = converter.unstructure(dt)
# '2024-01-15T10:30:00'
```

The unstructure hook signature is `func(value) -> unstructured_value`.

### `converter.register_structure_hook_func(predicate, func)` -- Predicate-Based Hooks

Register a structuring hook that applies to any type matching a predicate function. This is powerful for handling families of types (e.g., all Enums, all subclasses of a base).

```python
import cattrs
from enum import Enum

converter = cattrs.Converter()

# Handle all Enum subclasses
def is_enum(type_):
    try:
        return issubclass(type_, Enum)
    except TypeError:
        return False

def structure_enum(val, type_):
    return type_(val)

converter.register_structure_hook_func(is_enum, structure_enum)
```

> **Note**: In older cattrs versions this was called `register_structure_hook_strategy`. The predicate-based method is `register_structure_hook_func`. The predicate function receives the type and must return `True`/`False`.

### `converter.register_unstructure_hook_func(predicate, func)` -- Predicate-Based Unstructure Hooks

The unstructuring equivalent of predicate-based hooks:

```python
import cattrs
from enum import Enum

converter = cattrs.Converter()

def is_enum(type_):
    try:
        return issubclass(type_, Enum)
    except TypeError:
        return False

def unstructure_enum(val):
    return val.value

converter.register_unstructure_hook_func(is_enum, unstructure_enum)
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

## Preconf Converters

cattrs provides pre-configured converters for specific serialization formats. These converters come with hooks already registered for common types like `datetime`, `UUID`, `bytes`, `set`, `frozenset`, and more -- types that the base format cannot represent natively.

**Key concept**: Each preconf converter knows the quirks of its target format. For example, JSON cannot represent `datetime` natively, so `cattrs.preconf.json` registers hooks to convert `datetime` to/from ISO 8601 strings. BSON *can* represent `datetime` natively, so `cattrs.preconf.bson` leaves it alone.

### `cattrs.preconf.json` -- JSON-Compatible Converter

Produces output compatible with the standard `json` module. Handles `datetime`, `date`, `UUID`, `set`, `frozenset`, `bytes`, `Decimal`, and other types that are not natively JSON-serializable.

```python
from cattrs.preconf.json import make_converter
import json
from datetime import datetime
from uuid import UUID

converter = make_converter()

import attr

@attr.s(auto_attribs=True)
class Event:
    id: UUID
    name: str
    timestamp: datetime

event = Event(
    id=UUID("12345678-1234-5678-1234-567812345678"),
    name="deploy",
    timestamp=datetime(2024, 1, 15, 10, 30),
)

# Unstructure to JSON-compatible dict
raw = converter.unstructure(event)
# {'id': '12345678-1234-5678-1234-567812345678', 'name': 'deploy',
#  'timestamp': '2024-01-15T10:30:00'}

# All values are JSON-serializable
json_str = json.dumps(raw)

# Structure back from JSON-parsed dict
parsed = json.loads(json_str)
event2 = converter.structure(parsed, Event)
assert event2 == event
```

### `cattrs.preconf.orjson` -- orjson-Optimized Converter

Optimized for [orjson](https://github.com/ijl/orjson), the fastest JSON library for Python. orjson natively handles `datetime`, `UUID`, `bytes`, and more, so this converter defers to orjson's built-in support where possible.

```python
from cattrs.preconf.orjson import make_converter
import orjson

converter = make_converter()

# Unstructure and serialize in one step
raw = converter.unstructure(event)
json_bytes = orjson.dumps(raw)  # Returns bytes, not str

# Deserialize and structure
parsed = orjson.loads(json_bytes)
event2 = converter.structure(parsed, Event)
```

**Performance note**: orjson + cattrs preconf is typically the fastest option for JSON serialization of typed Python objects.

### `cattrs.preconf.msgpack` -- msgpack Format

For [msgpack](https://github.com/msgpack/msgpack-python) (MessagePack), a compact binary serialization format.

```python
from cattrs.preconf.msgpack import make_converter
import msgpack

converter = make_converter()

raw = converter.unstructure(event)
packed = msgpack.packb(raw)

unpacked = msgpack.unpackb(packed)
event2 = converter.structure(unpacked, Event)
```

### `cattrs.preconf.bson` -- BSON Format

For [bson](https://pymongo.readthedocs.io/en/stable/api/bson/) (Binary JSON, used by MongoDB). BSON natively supports `datetime`, `bytes`, `ObjectId`, etc.

```python
from cattrs.preconf.bson import make_converter
import bson

converter = make_converter()

raw = converter.unstructure(event)
bson_bytes = bson.encode(raw)

decoded = bson.decode(bson_bytes)
event2 = converter.structure(decoded, Event)
```

### `cattrs.preconf.tomlkit` -- TOML Format

For [tomlkit](https://github.com/sdispater/tomlkit) (TOML serialization).

```python
from cattrs.preconf.tomlkit import make_converter
import tomlkit

converter = make_converter()

raw = converter.unstructure(config_obj)
toml_str = tomlkit.dumps(raw)

parsed = tomlkit.loads(toml_str)
config = converter.structure(parsed, ConfigType)
```

### `cattrs.preconf.pyyaml` -- YAML Format

For [pyyaml](https://pyyaml.org/) (YAML serialization).

```python
from cattrs.preconf.pyyaml import make_converter
import yaml

converter = make_converter()

raw = converter.unstructure(config_obj)
yaml_str = yaml.dump(raw)

parsed = yaml.safe_load(yaml_str)
config = converter.structure(parsed, ConfigType)
```

### Preconf Summary Table

| Module | Format Library | Handles natively | Best for |
|---|---|---|---|
| `cattrs.preconf.json` | `json` (stdlib) | Strings, ints, floats, bools, None, lists, dicts | Standard JSON APIs |
| `cattrs.preconf.orjson` | `orjson` | + datetime, UUID, bytes | High-performance JSON |
| `cattrs.preconf.msgpack` | `msgpack` | + bytes, binary | Compact binary protocols |
| `cattrs.preconf.bson` | `bson` | + datetime, bytes, ObjectId | MongoDB, document stores |
| `cattrs.preconf.tomlkit` | `tomlkit` | + datetime, date, time | Configuration files |
| `cattrs.preconf.pyyaml` | `pyyaml` | Strings, ints, floats, bools, None, lists, dicts | Configuration files |

---

## Strategy System

Strategies are higher-level patterns that configure a converter for common use cases. They are applied to an existing converter and modify its hooks.

### `cattrs.strategies.include_subclasses` -- Polymorphic Structuring

Automatically handles structuring into the correct subclass by examining all subclasses of a base type.

```python
import attr
import cattrs
from cattrs.strategies import include_subclasses

@attr.s(auto_attribs=True)
class Shape:
    x: float
    y: float

@attr.s(auto_attribs=True)
class Circle(Shape):
    radius: float

@attr.s(auto_attribs=True)
class Rectangle(Shape):
    width: float
    height: float

converter = cattrs.Converter()
include_subclasses(Shape, converter)

# cattrs will try all subclasses and pick the one that fits
circle = converter.structure(
    {"x": 0, "y": 0, "radius": 5.0},
    Shape,
)
assert isinstance(circle, Circle)

rect = converter.structure(
    {"x": 1, "y": 2, "width": 10, "height": 20},
    Shape,
)
assert isinstance(rect, Rectangle)
```

**Caveat**: `include_subclasses` works by trying each subclass in turn. If subclasses have overlapping fields, it may pick the wrong one. For unambiguous dispatch, use `configure_tagged_union` instead.

### `cattrs.strategies.configure_tagged_union` -- Discriminated Unions

The recommended approach for polymorphism. Uses a "type tag" field (discriminator) in the data to determine which class to structure into.

```python
import attr
import cattrs
from cattrs.strategies import configure_tagged_union

@attr.s(auto_attribs=True)
class Circle:
    radius: float

@attr.s(auto_attribs=True)
class Rectangle:
    width: float
    height: float

converter = cattrs.Converter()

# Configure a tagged union: the "type" field determines the class
configure_tagged_union(
    Circle | Rectangle,  # or Union[Circle, Rectangle]
    converter,
    tag_name="type",  # field name in the data
    tag_generator=lambda t: t.__name__.lower(),  # how to map class -> tag
)

# Structure using the tag
circle = converter.structure(
    {"type": "circle", "radius": 5.0},
    Circle | Rectangle,
)
assert isinstance(circle, Circle)
assert circle.radius == 5.0

rect = converter.structure(
    {"type": "rectangle", "width": 10, "height": 20},
    Circle | Rectangle,
)
assert isinstance(rect, Rectangle)
```

**Configuration options:**

- `tag_name`: The key in the dict that holds the type discriminator (default: `"type"`).
- `tag_generator`: A callable `(type) -> str` that generates the tag value for each class. Default uses the class name.
- `default`: A default type to use if the tag is missing or unknown.

### `cattrs.strategies.use_class_methods` -- Using from_dict/to_dict Methods

Delegates structuring/unstructuring to methods on the class itself. Useful when classes already have their own serialization logic.

```python
import attr
import cattrs
from cattrs.strategies import use_class_methods

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

    @classmethod
    def _from_dict(cls, data: dict) -> "User":
        # Custom deserialization logic
        return cls(
            name=data["full_name"],
            age=data["years_old"],
        )

    def _to_dict(self) -> dict:
        # Custom serialization logic
        return {
            "full_name": self.name,
            "years_old": self.age,
        }

converter = cattrs.Converter()
use_class_methods(
    converter,
    structure_method_name="_from_dict",
    unstructure_method_name="_to_dict",
)

user = converter.structure({"full_name": "Alice", "years_old": 30}, User)
assert user.name == "Alice"

raw = converter.unstructure(user)
assert raw == {"full_name": "Alice", "years_old": 30}
```

---

## GenConverter

`GenConverter` is cattrs' code-generating converter. Instead of using generic dispatch logic at runtime, it generates specialized Python functions for each type at hook-creation time. The generated code is equivalent to what you would write by hand for maximum performance.

### How It Works

When a `GenConverter` encounters a new type for the first time, it:

1. Inspects the type's fields (attrs attributes, dataclass fields, etc.).
2. Generates Python source code for a specialized structuring/unstructuring function.
3. Compiles the function using `exec()`.
4. Caches the compiled function as the hook for that type.

Subsequent calls for the same type use the cached compiled function directly.

### When to Use GenConverter

- **Performance-critical paths**: The generated code eliminates dictionary lookups and dynamic dispatch overhead.
- **Field renaming/omitting**: `GenConverter` supports `override()` for renaming and omitting fields, which the base `Converter` does not.
- **Complex nested structures**: Generated code inlines the conversion of nested types for fewer function calls.

### Performance

Typical benchmarks show `GenConverter` is 2-5x faster than the base `Converter` for structuring/unstructuring attrs classes. The improvement is most significant for classes with many fields and deeply nested structures.

```python
import cattrs
import attr
from cattrs.gen import override

converter = cattrs.GenConverter()

@attr.s(auto_attribs=True)
class LargeModel:
    field_1: str
    field_2: int
    field_3: float
    field_4: bool
    # ... many fields ...

# GenConverter generates optimized code for this type
obj = converter.structure({"field_1": "a", "field_2": 1, "field_3": 2.0, "field_4": True}, LargeModel)
```

### GenConverter with `override()`

The key differentiator. `override()` lets you customize how individual fields are handled during structuring and unstructuring:

```python
import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int
    internal_id: str = ""

converter = cattrs.GenConverter()

# Rename 'name' to 'userName' in the unstructured output
# Omit 'internal_id' from the output entirely
converter.register_unstructure_hook(
    User,
    make_dict_unstructure_fn(
        User,
        converter,
        name=override(rename="userName"),
        internal_id=override(omit=True),
    ),
)

user = User(name="Alice", age=30, internal_id="secret-123")
raw = converter.unstructure(user)
# {'userName': 'Alice', 'age': 30}
# Note: 'internal_id' is omitted, 'name' is renamed to 'userName'

# For structuring, register the inverse
converter.register_structure_hook(
    User,
    make_dict_structure_fn(
        User,
        converter,
        name=override(rename="userName"),
    ),
)

user2 = converter.structure({"userName": "Bob", "age": 25}, User)
assert user2.name == "Bob"
```

---

## Type Support

cattrs supports a comprehensive set of Python types for structuring and unstructuring.

### attrs Classes

First-class support. attrs classes are the primary use case for cattrs.

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class Point:
    x: float
    y: float

@attr.define
class Point2:
    x: float
    y: float

# Both styles work
p = cattrs.structure({"x": 1.0, "y": 2.0}, Point)
p2 = cattrs.structure({"x": 1.0, "y": 2.0}, Point2)
```

### Dataclasses

Full support, equivalent to attrs classes.

```python
from dataclasses import dataclass
import cattrs

@dataclass
class Point:
    x: float
    y: float

p = cattrs.structure({"x": 1.0, "y": 2.0}, Point)
raw = cattrs.unstructure(p)
```

### `Optional[T]` and `Union[T1, T2, ...]`

```python
from typing import Optional, Union
import cattrs

# Optional -- structures None or the inner type
cattrs.structure(None, Optional[int])   # None
cattrs.structure(42, Optional[int])     # 42

# Union -- cattrs needs disambiguation hints for non-trivial unions
# Simple cases (Union with None, i.e. Optional) work automatically
cattrs.structure(None, Optional[str])  # None
cattrs.structure("hi", Optional[str])  # 'hi'
```

**Important**: For `Union[TypeA, TypeB]` (where neither is `None`), cattrs cannot automatically determine which type to use. You must register a custom hook or use `configure_tagged_union`. See [Gotchas](#2-union-disambiguation).

### `Literal`

```python
from typing import Literal
import cattrs

cattrs.structure("red", Literal["red", "green", "blue"])  # 'red'
cattrs.structure("purple", Literal["red", "green", "blue"])  # raises
```

### Collection Types: `list`, `dict`, `set`, `tuple`, `frozenset`

```python
import cattrs

# Lists
cattrs.structure([1, 2, 3], list[int])                # [1, 2, 3]
cattrs.structure(["1", "2", "3"], list[int])           # [1, 2, 3] (coerced)

# Dicts
cattrs.structure({"a": 1}, dict[str, int])             # {'a': 1}

# Sets
cattrs.structure([1, 2, 3], set[int])                  # {1, 2, 3}

# Tuples (heterogeneous)
cattrs.structure([1, "a", 3.0], tuple[int, str, float])  # (1, 'a', 3.0)

# Tuples (homogeneous)
cattrs.structure([1, 2, 3], tuple[int, ...])           # (1, 2, 3)

# Frozensets
cattrs.structure([1, 2, 3], frozenset[int])            # frozenset({1, 2, 3})
```

### `datetime`, `date`, `UUID`, `Path`, `Enum`

These require custom hooks or a preconf converter. The base `Converter` does **not** handle these automatically.

```python
import cattrs
from datetime import datetime, date
from uuid import UUID
from pathlib import Path
from enum import Enum

# Using a preconf converter (recommended)
from cattrs.preconf.json import make_converter
converter = make_converter()

# datetime -- structured from ISO strings
dt = converter.structure("2024-01-15T10:30:00", datetime)

# UUID -- structured from string
uid = converter.structure("12345678-1234-5678-1234-567812345678", UUID)

# Or register hooks manually on a plain Converter
converter = cattrs.Converter()
converter.register_structure_hook(datetime, lambda v, _: datetime.fromisoformat(v))
converter.register_unstructure_hook(datetime, lambda v: v.isoformat())

converter.register_structure_hook(UUID, lambda v, _: UUID(v))
converter.register_unstructure_hook(UUID, lambda v: str(v))

converter.register_structure_hook(Path, lambda v, _: Path(v))
converter.register_unstructure_hook(Path, lambda v: str(v))

# Enums
class Color(Enum):
    RED = "red"
    GREEN = "green"
    BLUE = "blue"

# Enums are handled by the base Converter (structured by value)
cattrs.structure("red", Color)    # Color.RED
cattrs.unstructure(Color.RED)     # 'red'
```

### `typing.TypedDict`

```python
from typing import TypedDict
import cattrs

class UserDict(TypedDict):
    name: str
    age: int

result = cattrs.structure({"name": "Alice", "age": 30}, UserDict)
# {'name': 'Alice', 'age': 30}  -- it IS a dict, but type-checked
```

### `typing.NamedTuple`

```python
from typing import NamedTuple
import cattrs

class Point(NamedTuple):
    x: float
    y: float

p = cattrs.structure({"x": 1.0, "y": 2.0}, Point)
# Point(x=1.0, y=2.0)

# Can also structure from a sequence
p2 = cattrs.structure([1.0, 2.0], Point)
```

### Forward References

cattrs handles forward references (string annotations) when the referenced type is available at structuring time:

```python
from __future__ import annotations
import attr
import cattrs

@attr.s(auto_attribs=True)
class TreeNode:
    value: int
    children: list[TreeNode]  # Forward reference (due to __future__.annotations)

data = {
    "value": 1,
    "children": [
        {"value": 2, "children": []},
        {"value": 3, "children": [
            {"value": 4, "children": []},
        ]},
    ],
}

node = cattrs.structure(data, TreeNode)
assert node.value == 1
assert node.children[1].children[0].value == 4
```

### Generic Types (`T`, `List[T]`)

cattrs supports generic attrs classes with type parameters:

```python
from typing import TypeVar, Generic, List
import attr
import cattrs

T = TypeVar("T")

@attr.s(auto_attribs=True)
class Container(Generic[T]):
    value: T
    items: List[T]

# Structure with a concrete type parameter
int_container = cattrs.structure(
    {"value": 42, "items": [1, 2, 3]},
    Container[int],
)
assert int_container.value == 42
assert int_container.items == [1, 2, 3]

str_container = cattrs.structure(
    {"value": "hello", "items": ["a", "b"]},
    Container[str],
)
assert str_container.value == "hello"
```

---

## Detailed Hooks

### The Hook System

cattrs uses a hook-based architecture. When you call `converter.structure(data, SomeType)`, the converter looks up the registered hook for `SomeType` and calls it. Hooks are matched in this priority order:

1. **Exact type hooks** registered via `register_structure_hook(type, func)`.
2. **Predicate hooks** registered via `register_structure_hook_func(predicate, func)`, checked in reverse registration order (last registered = highest priority).
3. **Built-in hooks** for known types (attrs classes, dataclasses, lists, dicts, etc.).

### `make_dict_structure_fn` -- Generate a Structuring Function

Generates a specialized structuring function for an attrs class or dataclass. This is what `GenConverter` uses internally, but you can use it directly for fine-grained control.

```python
import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, override

@attr.s(auto_attribs=True)
class User:
    name: str
    email: str
    age: int = 0

converter = cattrs.Converter()

# Generate a structuring function with overrides
structure_user = make_dict_structure_fn(
    User,
    converter,
    # Rename: the input dict has "user_name", map it to the "name" field
    name=override(rename="user_name"),
    # Omit: don't try to read "age" from the dict; use the default
    age=override(omit=True),
)

converter.register_structure_hook(User, structure_user)

user = converter.structure({"user_name": "Alice", "email": "alice@example.com"}, User)
assert user.name == "Alice"
assert user.age == 0  # default, since we omitted it
```

### `make_dict_unstructure_fn` -- Generate an Unstructuring Function

The unstructuring counterpart:

```python
import attr
import cattrs
from cattrs.gen import make_dict_unstructure_fn, override

@attr.s(auto_attribs=True)
class User:
    name: str
    email: str
    password_hash: str = ""

converter = cattrs.Converter()

# Generate an unstructuring function
unstructure_user = make_dict_unstructure_fn(
    User,
    converter,
    # Rename 'name' -> 'userName' in output
    name=override(rename="userName"),
    # Omit password_hash from output
    password_hash=override(omit=True),
)

converter.register_unstructure_hook(User, unstructure_user)

user = User(name="Alice", email="alice@example.com", password_hash="abc123")
raw = converter.unstructure(user)
# {'userName': 'Alice', 'email': 'alice@example.com'}
# password_hash is omitted, name is renamed
```

### `override()` -- Field-Level Overrides

The `override()` function creates field-level customization for generated structuring/unstructuring functions.

```python
from cattrs.gen import override

# Available options:
override(
    rename="new_key_name",  # Rename the field's key in the dict
    omit=True,              # Skip this field entirely
    omit_if_default=True,   # Skip if the value equals the field's default
    struct_hook=my_func,    # Use a custom structuring function for this field
    unstruct_hook=my_func,  # Use a custom unstructuring function for this field
)
```

**`omit_if_default`** is particularly useful for compact serialization:

```python
import attr
import cattrs
from cattrs.gen import make_dict_unstructure_fn, override

@attr.s(auto_attribs=True)
class Config:
    host: str = "localhost"
    port: int = 8080
    debug: bool = False
    workers: int = 4

converter = cattrs.GenConverter()

converter.register_unstructure_hook(
    Config,
    make_dict_unstructure_fn(
        Config,
        converter,
        host=override(omit_if_default=True),
        port=override(omit_if_default=True),
        debug=override(omit_if_default=True),
        workers=override(omit_if_default=True),
    ),
)

# Only non-default values appear in output
config = Config(debug=True)
raw = converter.unstructure(config)
# {'debug': True}  -- host, port, workers are all defaults, so omitted

config2 = Config(host="prod.example.com", workers=16)
raw2 = converter.unstructure(config2)
# {'host': 'prod.example.com', 'workers': 16}
```

---

## Gotchas and Common Mistakes

### 1. Hook Registration Order Matters

Predicate-based hooks (`register_structure_hook_func`) are checked in **reverse registration order** -- the last one registered is checked first. If two predicates both match a type, the one registered later wins.

```python
import cattrs

converter = cattrs.Converter()

# Registered first -- lower priority
converter.register_structure_hook_func(
    lambda t: hasattr(t, "__attrs_attrs__"),
    lambda v, t: f"attrs hook: {v}",
)

# Registered second -- higher priority (checked first)
converter.register_structure_hook_func(
    lambda t: hasattr(t, "__attrs_attrs__"),
    lambda v, t: f"override hook: {v}",
)

# The second hook wins because it was registered later
```

**Exact type hooks** (`register_structure_hook(type, func)`) always take precedence over predicate hooks, regardless of registration order.

**Best practice**: Register your most specific hooks last, or use exact type hooks when possible to avoid ambiguity.

### 2. Union Disambiguation (cattrs Cannot Always Auto-Detect)

cattrs **cannot** automatically disambiguate `Union[TypeA, TypeB]` when both types are attrs classes or dicts with overlapping structures. This is one of the most common sources of confusion.

```python
import attr
import cattrs
from typing import Union

@attr.s(auto_attribs=True)
class Cat:
    name: str
    indoor: bool

@attr.s(auto_attribs=True)
class Dog:
    name: str
    breed: str

# THIS WILL FAIL or produce incorrect results:
# cattrs.structure({"name": "Rex", "breed": "Lab"}, Union[Cat, Dog])
```

**Solutions:**

**Option A**: Use `configure_tagged_union` (recommended):

```python
from cattrs.strategies import configure_tagged_union

converter = cattrs.Converter()
configure_tagged_union(Union[Cat, Dog], converter, tag_name="_type")

data = {"_type": "Dog", "name": "Rex", "breed": "Lab"}
pet = converter.structure(data, Union[Cat, Dog])
assert isinstance(pet, Dog)
```

**Option B**: Register a custom union hook with disambiguation logic:

```python
converter = cattrs.Converter()

def structure_pet(data, _):
    if "breed" in data:
        return converter.structure(data, Dog)
    return converter.structure(data, Cat)

converter.register_structure_hook(Union[Cat, Dog], structure_pet)
```

**Option C**: Use `cattrs.disambiguators.create_default_dis_func` for automatic field-based disambiguation (works when types have unique required fields):

```python
# If Cat has 'indoor' and Dog has 'breed', cattrs can use those
# unique fields to tell them apart, but only if they are truly unique.
```

### 3. Missing Fields vs None Fields

cattrs distinguishes between a field being absent from the input dict and it being present with a `None` value.

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int = 0  # Has a default

# Missing field: uses the default
user = cattrs.structure({"name": "Alice"}, User)
assert user.age == 0  # Default applied

# Explicit None: raises (int is not Optional)
# cattrs.structure({"name": "Alice", "age": None}, User)
# ClassValidationError: expected int, got None
```

If a field should accept `None`, declare it as `Optional`:

```python
from typing import Optional

@attr.s(auto_attribs=True)
class User:
    name: str
    age: Optional[int] = None

# Now None is accepted
user = cattrs.structure({"name": "Alice", "age": None}, User)
assert user.age is None
```

### 4. Converter Instance Sharing vs Isolation

Do **not** share a single global `Converter` across unrelated subsystems if they need different serialization rules. Hooks registered on a converter affect all structuring/unstructuring done through that converter.

```python
import cattrs

# BAD: One converter with conflicting requirements
converter = cattrs.Converter()

# The API layer wants camelCase
# The database layer wants snake_case
# Both registering hooks on the same converter will conflict!

# GOOD: Separate converters
api_converter = cattrs.Converter()
db_converter = cattrs.Converter()
# Register API hooks on api_converter, DB hooks on db_converter
```

**Rule of thumb**: Create one converter per serialization boundary (API, database, cache, message queue, etc.).

### 5. Performance: GenConverter vs Converter

| Scenario | `Converter` | `GenConverter` |
|---|---|---|
| Simple attrs class (5 fields) | Baseline | ~2-3x faster |
| Deeply nested (3+ levels) | Baseline | ~3-5x faster |
| First-time structuring | Faster (no codegen) | Slower (generates code) |
| Repeated structuring | Baseline | Much faster |
| Field renaming/omitting | Not supported | Supported via `override()` |

**Guidance**: Use `GenConverter` in production for hot paths. Use plain `Converter` for scripts, tests, or when you need maximum flexibility. In modern cattrs versions, the base `Converter` also uses some code generation for attrs/dataclass types, so the gap is smaller than in older versions.

### 6. Recursive / Self-Referential Types

cattrs handles self-referential types, but you may need to be explicit about converter usage:

```python
from __future__ import annotations
import attr
import cattrs

@attr.s(auto_attribs=True)
class TreeNode:
    value: int
    children: list[TreeNode]

# This works out of the box
data = {"value": 1, "children": [{"value": 2, "children": []}]}
node = cattrs.structure(data, TreeNode)
```

However, if you use `GenConverter` with custom hooks and the type is self-referential, make sure you register the hook **before** any structuring call, as the generated code needs to reference the converter's hook for the recursive type.

### 7. Extra Keys in Input Dicts

By default, the base `Converter` ignores extra keys in the input dict. `GenConverter` also ignores them by default. If you want strict mode (reject unknown keys), you need to implement it yourself:

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

# Extra key 'email' is silently ignored
user = cattrs.structure({"name": "Alice", "age": 30, "email": "alice@example.com"}, User)
assert user.name == "Alice"
# No 'email' attribute on User -- it was just dropped
```

### 8. attrs Validators Run During Structuring

If your attrs class has validators, they run when the class is instantiated during structuring. This can cause structuring to fail even when the data types are correct:

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class PositiveInt:
    value: int = attr.ib(validator=attr.validators.gt(0))

cattrs.structure({"value": 5}, PositiveInt)   # OK
cattrs.structure({"value": -1}, PositiveInt)  # Raises! Validator fails
```

### 9. `ClassValidationError` for Detailed Errors

cattrs raises `cattrs.errors.ClassValidationError` (a subclass of `BaseException`) that contains detailed per-field error information:

```python
import attr
import cattrs
from cattrs.errors import ClassValidationError

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

try:
    cattrs.structure({"name": "Alice", "age": "not_a_number"}, User)
except ClassValidationError as e:
    print(e.exceptions)  # List of per-field exceptions
    # Detailed error information about which field failed and why
```

---

## Complete Code Examples

### Example 1: Basic Structure and Unstructure

```python
"""
Basic structuring and unstructuring with attrs classes.
"""

import attr
import cattrs

@attr.s(auto_attribs=True)
class Address:
    street: str
    city: str
    zip_code: str

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int
    address: Address

# Structure from a nested dict (e.g., JSON API response)
raw = {
    "name": "Alice",
    "age": 30,
    "address": {
        "street": "123 Main St",
        "city": "Springfield",
        "zip_code": "62701",
    },
}

user = cattrs.structure(raw, User)
assert isinstance(user, User)
assert isinstance(user.address, Address)
assert user.name == "Alice"
assert user.address.city == "Springfield"

# Unstructure back to a plain dict (e.g., for JSON serialization)
raw_back = cattrs.unstructure(user)
assert raw_back == raw
assert isinstance(raw_back, dict)
assert isinstance(raw_back["address"], dict)
```

### Example 2: Custom Hooks for Dates and Enums

```python
"""
Register custom hooks for datetime and Enum types.
"""

import attr
import cattrs
from datetime import datetime, date
from enum import Enum

class Priority(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@attr.s(auto_attribs=True)
class Task:
    title: str
    priority: Priority
    due_date: date
    created_at: datetime

# Create a converter with custom hooks
converter = cattrs.Converter()

# Date: structured from "YYYY-MM-DD" strings
converter.register_structure_hook(date, lambda v, _: date.fromisoformat(v))
converter.register_unstructure_hook(date, lambda v: v.isoformat())

# Datetime: structured from ISO format strings
converter.register_structure_hook(datetime, lambda v, _: datetime.fromisoformat(v))
converter.register_unstructure_hook(datetime, lambda v: v.isoformat())

# Enums are handled automatically by cattrs, but you can customize:
# converter.register_structure_hook(Priority, lambda v, _: Priority(v))
# converter.register_unstructure_hook(Priority, lambda v: v.value)

# Structure from raw data
raw = {
    "title": "Write docs",
    "priority": "high",
    "due_date": "2024-03-01",
    "created_at": "2024-01-15T10:30:00",
}

task = converter.structure(raw, Task)
assert task.priority == Priority.HIGH
assert task.due_date == date(2024, 3, 1)
assert task.created_at == datetime(2024, 1, 15, 10, 30)

# Unstructure back
raw_back = converter.unstructure(task)
assert raw_back["due_date"] == "2024-03-01"
assert raw_back["created_at"] == "2024-01-15T10:30:00"
assert raw_back["priority"] == "high"
```

### Example 3: Tagged Unions for Polymorphism

```python
"""
Using configure_tagged_union for discriminated polymorphic deserialization.
"""

import attr
import cattrs
from typing import Union, List
from cattrs.strategies import configure_tagged_union

@attr.s(auto_attribs=True)
class TextBlock:
    content: str
    bold: bool = False

@attr.s(auto_attribs=True)
class ImageBlock:
    url: str
    alt_text: str = ""

@attr.s(auto_attribs=True)
class CodeBlock:
    code: str
    language: str = "python"

# The union of all block types
Block = Union[TextBlock, ImageBlock, CodeBlock]

@attr.s(auto_attribs=True)
class Document:
    title: str
    blocks: List[Block]

# Configure the converter with tagged union discrimination
converter = cattrs.Converter()
configure_tagged_union(
    Block,
    converter,
    tag_name="block_type",
    tag_generator=lambda t: t.__name__,  # Use class name as the tag
)

# Raw data with type tags
raw_doc = {
    "title": "My Document",
    "blocks": [
        {"block_type": "TextBlock", "content": "Hello, world!", "bold": True},
        {"block_type": "ImageBlock", "url": "https://example.com/img.png", "alt_text": "Example"},
        {"block_type": "CodeBlock", "code": "print('hi')", "language": "python"},
        {"block_type": "TextBlock", "content": "Goodbye!"},
    ],
}

doc = converter.structure(raw_doc, Document)
assert isinstance(doc.blocks[0], TextBlock)
assert isinstance(doc.blocks[1], ImageBlock)
assert isinstance(doc.blocks[2], CodeBlock)
assert isinstance(doc.blocks[3], TextBlock)
assert doc.blocks[0].bold is True
assert doc.blocks[1].url == "https://example.com/img.png"
assert doc.blocks[2].language == "python"
```

### Example 4: Nested attrs Classes with Lists and Optionals

```python
"""
Structuring deeply nested data with collections and optional fields.
"""

import attr
import cattrs
from typing import List, Optional

@attr.s(auto_attribs=True)
class Ingredient:
    name: str
    amount: str
    optional: bool = False

@attr.s(auto_attribs=True)
class Step:
    instruction: str
    duration_minutes: Optional[int] = None

@attr.s(auto_attribs=True)
class Recipe:
    name: str
    servings: int
    ingredients: List[Ingredient]
    steps: List[Step]
    notes: Optional[str] = None

raw = {
    "name": "Pancakes",
    "servings": 4,
    "ingredients": [
        {"name": "flour", "amount": "2 cups"},
        {"name": "milk", "amount": "1.5 cups"},
        {"name": "eggs", "amount": "2"},
        {"name": "blueberries", "amount": "1 cup", "optional": True},
    ],
    "steps": [
        {"instruction": "Mix dry ingredients", "duration_minutes": 5},
        {"instruction": "Add wet ingredients", "duration_minutes": 3},
        {"instruction": "Cook on griddle", "duration_minutes": 15},
        {"instruction": "Serve warm"},  # duration_minutes is None (missing)
    ],
    "notes": None,  # Explicitly null
}

recipe = cattrs.structure(raw, Recipe)

assert recipe.name == "Pancakes"
assert len(recipe.ingredients) == 4
assert recipe.ingredients[3].optional is True
assert recipe.steps[2].duration_minutes == 15
assert recipe.steps[3].duration_minutes is None
assert recipe.notes is None

# Round-trip
raw_back = cattrs.unstructure(recipe)
assert raw_back["ingredients"][0]["name"] == "flour"
```

### Example 5: Preconf Converter for JSON APIs

```python
"""
Using the preconf JSON converter for a real-world API client.
"""

import attr
import json
from datetime import datetime
from uuid import UUID
from typing import List, Optional
from cattrs.preconf.json import make_converter

# Create a JSON-optimized converter
converter = make_converter()

@attr.s(auto_attribs=True)
class Author:
    id: UUID
    name: str

@attr.s(auto_attribs=True)
class Comment:
    id: UUID
    author: Author
    body: str
    created_at: datetime

@attr.s(auto_attribs=True)
class BlogPost:
    id: UUID
    title: str
    body: str
    author: Author
    comments: List[Comment]
    published_at: Optional[datetime] = None

# Simulate an API response (already parsed from JSON)
api_response = {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "Getting Started with cattrs",
    "body": "cattrs is a powerful serialization library...",
    "author": {
        "id": "11111111-2222-3333-4444-555555555555",
        "name": "Alice",
    },
    "comments": [
        {
            "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
            "author": {
                "id": "66666666-7777-8888-9999-000000000000",
                "name": "Bob",
            },
            "body": "Great article!",
            "created_at": "2024-01-15T14:30:00",
        },
    ],
    "published_at": "2024-01-15T10:00:00",
}

# Structure the API response into typed objects
post = converter.structure(api_response, BlogPost)

assert isinstance(post.id, UUID)
assert isinstance(post.author, Author)
assert isinstance(post.comments[0].created_at, datetime)
assert post.author.name == "Alice"
assert post.comments[0].body == "Great article!"

# Unstructure back to a JSON-compatible dict
raw = converter.unstructure(post)
json_str = json.dumps(raw, indent=2)

# The output is valid JSON with datetimes as ISO strings and UUIDs as strings
assert isinstance(raw["id"], str)
assert isinstance(raw["published_at"], str)
```

### Example 6: GenConverter with Overrides (Rename/Omit)

```python
"""
Using GenConverter with field renaming and omitting for API serialization.
"""

import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override

@attr.s(auto_attribs=True)
class UserProfile:
    first_name: str
    last_name: str
    email_address: str
    password_hash: str = ""
    is_admin: bool = False
    login_count: int = 0

converter = cattrs.GenConverter()

# Configure unstructuring: Python snake_case -> JSON camelCase
# Also omit sensitive and internal fields
converter.register_unstructure_hook(
    UserProfile,
    make_dict_unstructure_fn(
        UserProfile,
        converter,
        first_name=override(rename="firstName"),
        last_name=override(rename="lastName"),
        email_address=override(rename="emailAddress"),
        password_hash=override(omit=True),           # Never expose passwords
        is_admin=override(rename="isAdmin"),
        login_count=override(rename="loginCount", omit_if_default=True),
    ),
)

# Configure structuring: JSON camelCase -> Python snake_case
converter.register_structure_hook(
    UserProfile,
    make_dict_structure_fn(
        UserProfile,
        converter,
        first_name=override(rename="firstName"),
        last_name=override(rename="lastName"),
        email_address=override(rename="emailAddress"),
        is_admin=override(rename="isAdmin"),
        login_count=override(rename="loginCount"),
    ),
)

# Unstructure: Python -> API
user = UserProfile(
    first_name="Alice",
    last_name="Smith",
    email_address="alice@example.com",
    password_hash="bcrypt$$2b$12$...",
    is_admin=False,
    login_count=0,
)

api_output = converter.unstructure(user)
assert api_output == {
    "firstName": "Alice",
    "lastName": "Smith",
    "emailAddress": "alice@example.com",
    "isAdmin": False,
    # password_hash is omitted
    # login_count is omitted (default value + omit_if_default)
}

# Structure: API -> Python
api_input = {
    "firstName": "Bob",
    "lastName": "Jones",
    "emailAddress": "bob@example.com",
    "isAdmin": True,
    "loginCount": 42,
}

user2 = converter.structure(api_input, UserProfile)
assert user2.first_name == "Bob"
assert user2.last_name == "Jones"
assert user2.is_admin is True
assert user2.login_count == 42
assert user2.password_hash == ""  # Default, since not in input
```

### Example 7: Full Pipeline -- API Request to Database and Back

```python
"""
Complete example: an API layer with its own converter, a storage layer with
its own converter, and data flowing between them.
"""

import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override
from cattrs.preconf.json import make_converter as make_json_converter
from datetime import datetime, timezone
from uuid import UUID, uuid4
from typing import List, Optional

# --- Domain Model ---

@attr.s(auto_attribs=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: float

@attr.s(auto_attribs=True)
class Order:
    id: UUID
    customer_email: str
    items: List[OrderItem]
    total: float
    created_at: datetime
    notes: Optional[str] = None

# --- API Converter (camelCase, JSON-compatible) ---

api_converter = make_json_converter()

# Register camelCase renaming for the API layer
api_converter.register_unstructure_hook(
    OrderItem,
    make_dict_unstructure_fn(
        OrderItem,
        api_converter,
        product_id=override(rename="productId"),
        unit_price=override(rename="unitPrice"),
    ),
)
api_converter.register_structure_hook(
    OrderItem,
    make_dict_structure_fn(
        OrderItem,
        api_converter,
        product_id=override(rename="productId"),
        unit_price=override(rename="unitPrice"),
    ),
)

api_converter.register_unstructure_hook(
    Order,
    make_dict_unstructure_fn(
        Order,
        api_converter,
        customer_email=override(rename="customerEmail"),
        created_at=override(rename="createdAt"),
    ),
)
api_converter.register_structure_hook(
    Order,
    make_dict_structure_fn(
        Order,
        api_converter,
        customer_email=override(rename="customerEmail"),
        created_at=override(rename="createdAt"),
    ),
)

# --- Storage Converter (snake_case, internal format) ---

db_converter = make_json_converter()
# Uses defaults -- snake_case matches the attrs field names

# --- Simulate the flow ---

# 1. API request comes in (camelCase JSON)
api_request = {
    "customerEmail": "alice@example.com",
    "items": [
        {"productId": "SKU-001", "quantity": 2, "unitPrice": 29.99},
        {"productId": "SKU-042", "quantity": 1, "unitPrice": 49.99},
    ],
}

# 2. Structure the API input (partial -- we add server-side fields)
items = api_converter.structure(api_request["items"], List[OrderItem])
total = sum(item.unit_price * item.quantity for item in items)

order = Order(
    id=uuid4(),
    customer_email=api_request["customerEmail"],
    items=items,
    total=total,
    created_at=datetime.now(timezone.utc),
)

# 3. Store in database format (snake_case)
db_record = db_converter.unstructure(order)
assert "customer_email" in db_record  # snake_case
assert "created_at" in db_record

# 4. Load from database
loaded_order = db_converter.structure(db_record, Order)
assert loaded_order.id == order.id

# 5. Send API response (camelCase)
api_response = api_converter.unstructure(loaded_order)
assert "customerEmail" in api_response  # camelCase
assert "createdAt" in api_response

# The API response is JSON-serializable
import json
json_str = json.dumps(api_response)
```

### Example 8: Structuring with `include_subclasses`

```python
"""
Polymorphic structuring using include_subclasses.
"""

import attr
import cattrs
from cattrs.strategies import include_subclasses
from typing import List

@attr.s(auto_attribs=True)
class Notification:
    recipient: str
    message: str

@attr.s(auto_attribs=True)
class EmailNotification(Notification):
    subject: str

@attr.s(auto_attribs=True)
class SMSNotification(Notification):
    phone_number: str

@attr.s(auto_attribs=True)
class PushNotification(Notification):
    device_token: str
    badge_count: int = 0

converter = cattrs.Converter()
include_subclasses(Notification, converter)

# cattrs tries each subclass and picks the one whose fields match
data = [
    {"recipient": "alice", "message": "Hello!", "subject": "Greetings"},
    {"recipient": "bob", "message": "Your code is ready", "phone_number": "+1234567890"},
    {"recipient": "carol", "message": "New update", "device_token": "abc123", "badge_count": 3},
]

notifications = [converter.structure(d, Notification) for d in data]

assert isinstance(notifications[0], EmailNotification)
assert isinstance(notifications[1], SMSNotification)
assert isinstance(notifications[2], PushNotification)
assert notifications[0].subject == "Greetings"
assert notifications[2].badge_count == 3
```

### Example 9: Handling Complex Unions with Custom Disambiguation

```python
"""
Custom disambiguation for Union types where automatic detection is not possible.
"""

import attr
import cattrs
from typing import Union, List

@attr.s(auto_attribs=True)
class SuccessResponse:
    data: dict
    status_code: int = 200

@attr.s(auto_attribs=True)
class ErrorResponse:
    error: str
    error_code: int
    status_code: int

@attr.s(auto_attribs=True)
class RedirectResponse:
    location: str
    status_code: int = 302

ApiResponse = Union[SuccessResponse, ErrorResponse, RedirectResponse]

converter = cattrs.Converter()

def structure_api_response(data, _):
    """Disambiguate based on which keys are present."""
    if "error" in data:
        return converter.structure(data, ErrorResponse)
    elif "location" in data:
        return converter.structure(data, RedirectResponse)
    else:
        return converter.structure(data, SuccessResponse)

converter.register_structure_hook(ApiResponse, structure_api_response)

# Test disambiguation
responses = [
    {"data": {"user": "alice"}, "status_code": 200},
    {"error": "Not found", "error_code": 404, "status_code": 404},
    {"location": "https://example.com/new", "status_code": 301},
]

results = [converter.structure(r, ApiResponse) for r in responses]
assert isinstance(results[0], SuccessResponse)
assert isinstance(results[1], ErrorResponse)
assert isinstance(results[2], RedirectResponse)
assert results[1].error == "Not found"
```

### Example 10: Converter with Comprehensive Hook Setup

```python
"""
A production-ready converter configuration with hooks for all common types.
"""

import attr
import cattrs
from datetime import datetime, date, time, timedelta
from decimal import Decimal
from enum import Enum
from pathlib import Path, PurePosixPath
from uuid import UUID
from typing import Any

def make_production_converter() -> cattrs.GenConverter:
    """Create a fully configured converter for production use."""
    converter = cattrs.GenConverter()

    # --- datetime family ---
    converter.register_structure_hook(
        datetime, lambda v, _: datetime.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(datetime, lambda v: v.isoformat())

    converter.register_structure_hook(
        date, lambda v, _: date.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(date, lambda v: v.isoformat())

    converter.register_structure_hook(
        time, lambda v, _: time.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(time, lambda v: v.isoformat())

    converter.register_structure_hook(
        timedelta, lambda v, _: timedelta(seconds=v) if isinstance(v, (int, float)) else v
    )
    converter.register_unstructure_hook(timedelta, lambda v: v.total_seconds())

    # --- UUID ---
    converter.register_structure_hook(UUID, lambda v, _: UUID(v) if isinstance(v, str) else v)
    converter.register_unstructure_hook(UUID, lambda v: str(v))

    # --- Path ---
    converter.register_structure_hook(Path, lambda v, _: Path(v))
    converter.register_unstructure_hook(Path, lambda v: str(v))

    converter.register_structure_hook(PurePosixPath, lambda v, _: PurePosixPath(v))
    converter.register_unstructure_hook(PurePosixPath, lambda v: str(v))

    # --- Decimal ---
    converter.register_structure_hook(Decimal, lambda v, _: Decimal(str(v)))
    converter.register_unstructure_hook(Decimal, lambda v: str(v))

    # --- bytes ---
    import base64
    converter.register_structure_hook(
        bytes, lambda v, _: base64.b64decode(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(bytes, lambda v: base64.b64encode(v).decode("ascii"))

    return converter


# Usage
converter = make_production_converter()

@attr.s(auto_attribs=True)
class AuditEntry:
    id: UUID
    timestamp: datetime
    path: Path
    duration: timedelta
    payload_size: Decimal
    raw_data: bytes

entry = AuditEntry(
    id=UUID("12345678-1234-5678-1234-567812345678"),
    timestamp=datetime(2024, 1, 15, 10, 30, 0),
    path=Path("/var/log/app.log"),
    duration=timedelta(seconds=3.5),
    payload_size=Decimal("1024.50"),
    raw_data=b"hello world",
)

raw = converter.unstructure(entry)
assert raw == {
    "id": "12345678-1234-5678-1234-567812345678",
    "timestamp": "2024-01-15T10:30:00",
    "path": "/var/log/app.log",
    "duration": 3.5,
    "payload_size": "1024.50",
    "raw_data": "aGVsbG8gd29ybGQ=",
}

# Round-trip
entry2 = converter.structure(raw, AuditEntry)
assert entry2.id == entry.id
assert entry2.raw_data == b"hello world"
```

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

---

## References

- **GitHub**: https://github.com/python-attrs/cattrs
- **PyPI**: https://pypi.org/project/cattrs/
- **Documentation**: https://catt.rs/en/stable/
- **attrs documentation**: https://www.attrs.org/en/stable/ (cattrs' primary dependency)
- **Changelog**: https://catt.rs/en/stable/history.html
