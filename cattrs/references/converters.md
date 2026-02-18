# cattrs — Converters

> Part of the cattrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

1. [Custom Structuring Hooks](#converterregister_structure_hooktype-func----custom-structuring)
2. [Custom Unstructuring Hooks](#converterregister_unstructure_hooktype-func----custom-unstructuring)
3. [Predicate-Based Structure Hooks](#converterregister_structure_hook_funcpredicate-func----predicate-based-hooks)
4. [Predicate-Based Unstructure Hooks](#converterregister_unstructure_hook_funcpredicate-func----predicate-based-unstructure-hooks)
5. [GenConverter](#genconverter)
6. [GenConverter with override()](#genconverter-with-override)
7. [Detailed Hooks](#detailed-hooks)
8. [Preconf Converters](#preconf-converters)
9. [Strategy System](#strategy-system)

---

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
