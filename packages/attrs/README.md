# attrs

> Python classes without boilerplate.

`attrs` is the Python package that brings back the joy of writing classes by removing the tedium of implementing `__init__`, `__repr__`, `__eq__`, `__hash__`, and more. It was the direct inspiration for Python's built-in `dataclasses` module (PEP 557), but remains more powerful, more flexible, and more actively developed. **This is the foundation our entire stack is built on -- Rune, Transmute, and Enchant all depend on it.**

| Detail | Value |
|---|---|
| **PyPI** | [attrs](https://pypi.org/project/attrs/) |
| **Latest stable version** | **24.3.0** (verify at PyPI for newer releases) |
| **Python** | 3.7+ |
| **Source** | [github.com/python-attrs/attrs](https://github.com/python-attrs/attrs) |
| **Docs** | [www.attrs.org](https://www.attrs.org/en/stable/) |
| **License** | MIT |

---

## Table of Contents

1. [Installation](#installation)
2. [Overview and Philosophy](#overview-and-philosophy)
3. [Core API](#core-api)
4. [Validators](#validators)
5. [Converters](#converters)
6. [Slots vs Dict Classes](#slots-vs-dict-classes)
7. [Type Annotations](#type-annotations)
8. [on_setattr -- Controlling Attribute Setting](#on_setattr----controlling-attribute-setting)
9. [Aliases](#aliases)
10. [Comparison and Ordering](#comparison-and-ordering)
11. [attrs vs dataclasses](#attrs-vs-dataclasses)
12. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
13. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
pip install attrs
```

This installs the `attrs` package which provides both the modern `attrs` namespace and the legacy `attr` namespace. Always prefer the modern `attrs` namespace for new code.

For optional features:

```bash
# If you need cattrs for (de)serialization alongside attrs
pip install cattrs
```

---

## Overview and Philosophy

`attrs` lets you declare the structure of a class, and it generates all the dunder methods (`__init__`, `__repr__`, `__eq__`, `__hash__`, `__lt__`, etc.) for you. You focus on **what your class is**, not on the boilerplate of **how it behaves**.

```python
import attrs

@attrs.define
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
print(p)          # Point(x=1.0, y=2.0)
p == Point(1.0, 2.0)  # True
```

The package has two API generations:

| API | Decorator | Field | Notes |
|---|---|---|---|
| **Modern (recommended)** | `@attrs.define`, `@attrs.frozen`, `@attrs.mutable` | `attrs.field()` | Uses `slots=True` by default, type annotations for fields |
| **Legacy** | `@attr.s`, `@attr.attrs` | `attr.ib()`, `attr.attrib()` | Uses `dict` classes by default, supports `attr.ib()` field syntax |

This documentation covers the **modern API**. Use it for all new code.

---

## Core API

### `@attrs.define` -- Modern Mutable Classes

The primary decorator for creating attrs classes. Generates `__init__`, `__repr__`, `__eq__`, and uses **slots by default** for memory efficiency and faster attribute access.

```python
import attrs

@attrs.define
class User:
    name: str
    email: str
    age: int = 0
```

**Key parameters of `@attrs.define`:**

| Parameter | Default | Description |
|---|---|---|
| `slots` | `True` | Use `__slots__` instead of `__dict__` |
| `frozen` | `False` | Make instances immutable |
| `weakref_slot` | `True` | Add a `__weakref__` slot (only when `slots=True`) |
| `kw_only` | `False` | All fields become keyword-only in `__init__` |
| `eq` | `True` | Generate `__eq__` and `__ne__` |
| `order` | `False` | Generate `__lt__`, `__le__`, `__gt__`, `__ge__` |
| `hash` | `None` | Generate `__hash__` (auto-decided based on `eq` and `frozen`) |
| `repr` | `True` | Generate `__repr__` |
| `init` | `True` | Generate `__init__` |
| `on_setattr` | `attrs.setters.pipe(attrs.setters.convert, attrs.setters.validate)` | Hooks called on attribute setting |
| `match_args` | `True` | Generate `__match_args__` for Python 3.10+ pattern matching |

```python
import attrs

@attrs.define(order=True, kw_only=True)
class Priority:
    level: int
    label: str

# keyword-only:
p = Priority(level=1, label="high")

# ordering works:
Priority(level=1, label="a") < Priority(level=2, label="b")  # True
```

### `@attrs.frozen` -- Immutable Classes

Creates frozen (immutable) instances. Equivalent to `@attrs.define(frozen=True, on_setattr=attrs.setters.NO_OP)`. Frozen classes also get a `__hash__` by default, making them usable as dict keys and set members.

```python
import attrs

@attrs.frozen
class Color:
    red: int
    green: int
    blue: int

c = Color(255, 128, 0)
c.red = 100  # raises attrs.exceptions.FrozenInstanceError

# Hashable -- can be used as dict key
colors = {Color(255, 0, 0): "red", Color(0, 255, 0): "green"}
```

**Use `@attrs.frozen` when:**
- Values should never change after creation (domain value objects, configuration, messages).
- You need hashability for use in sets or as dict keys.
- You want thread-safe immutable data.

### `@attrs.mutable` -- Alias for `@attrs.define`

`attrs.mutable` is an explicit alias for `attrs.define`. Use it when you want to emphasize that a class is intentionally mutable, especially in a codebase that uses `@attrs.frozen` heavily.

```python
import attrs

@attrs.mutable  # same as @attrs.define
class Counter:
    value: int = 0

c = Counter()
c.value += 1  # allowed
```

### `attrs.field()` -- Field Configuration

Fine-grained control over individual fields. This is the modern replacement for `attr.ib()`.

```python
import attrs

@attrs.define
class Product:
    name: str
    price: float = attrs.field(validator=attrs.validators.gt(0))
    tags: list = attrs.field(factory=list)
    _internal: str = attrs.field(default="", repr=False, alias="_internal")
```

**Key parameters of `attrs.field()`:**

| Parameter | Description |
|---|---|
| `default` | Static default value (do NOT use mutable objects here -- use `factory` instead) |
| `factory` | Callable that produces the default value, called for each new instance |
| `validator` | Validator or list of validators to run on the value |
| `converter` | Callable to convert the value before setting |
| `repr` | Whether to include in `__repr__` (`True`/`False` or a callable for custom formatting) |
| `eq` | Whether to include in `__eq__` comparison |
| `hash` | Whether to include in `__hash__` |
| `init` | Whether to include in `__init__` |
| `alias` | Alternative name for the `__init__` parameter |
| `on_setattr` | Override per-field setattr behavior |
| `kw_only` | Make this specific field keyword-only |
| `metadata` | Dict of arbitrary user metadata (not used by attrs itself) |

```python
import attrs

@attrs.define
class Config:
    host: str = attrs.field(default="localhost")
    port: int = attrs.field(default=8080, validator=attrs.validators.instance_of(int))
    debug: bool = attrs.field(default=False, repr=False)
    headers: dict = attrs.field(factory=dict)
    _secret: str = attrs.field(default="", repr=False, eq=False, hash=False)
```

### `attrs.Factory(func, takes_self=False)` -- Lazy Defaults

Creates default values by calling a factory function. Essential for mutable defaults like lists, dicts, and sets. When `takes_self=True`, the factory receives the partially-initialized instance.

```python
import attrs

@attrs.define
class Team:
    name: str
    members: list = attrs.Factory(list)               # empty list per instance
    metadata: dict = attrs.Factory(dict)               # empty dict per instance
    created_at: float = attrs.Factory(time.time)       # current time at creation

# With takes_self -- factory receives the partially-initialized instance:
@attrs.define
class Node:
    name: str
    children: list = attrs.Factory(list)
    label: str = attrs.Factory(lambda self: self.name.upper(), takes_self=True)

n = Node("root")
n.label  # 'ROOT'
```

**Important:** `attrs.field(factory=list)` is shorthand for `attrs.field(default=attrs.Factory(list))`. Use the explicit `Factory` form when you need `takes_self=True`.

### `attrs.NOTHING` -- Sentinel Value

A sentinel object used internally by attrs to distinguish between "no value provided" and `None`. You encounter it when introspecting fields that have no default.

```python
import attrs

@attrs.define
class Item:
    name: str            # no default
    count: int = 0       # has a default

for field in attrs.fields(Item):
    if field.default is attrs.NOTHING:
        print(f"{field.name}: required (no default)")
    else:
        print(f"{field.name}: optional (default={field.default})")

# Output:
# name: required (no default)
# count: optional (default=0)
```

### `attrs.fields(cls)` -- Introspecting Fields

Returns a tuple of `attrs.Attribute` objects describing all fields of an attrs class. Useful for serialization, introspection, and metaprogramming.

```python
import attrs

@attrs.define
class Person:
    name: str
    age: int = 0

fields = attrs.fields(Person)
# (Attribute(name='name', ...), Attribute(name='age', ...))

fields.name   # Attribute for 'name' (named tuple access)
fields[0]     # Same thing, by index

# Each Attribute has:
field = fields.name
field.name        # 'name'
field.type        # str
field.default     # attrs.NOTHING (no default)
field.validator   # None or validator
field.converter   # None or converter
field.metadata    # {}
```

### `attrs.asdict()` and `attrs.astuple()` -- Serialization Helpers

Convert an attrs instance to a dict or tuple, recursively converting nested attrs instances.

```python
import attrs

@attrs.define
class Address:
    street: str
    city: str

@attrs.define
class Person:
    name: str
    age: int
    address: Address

p = Person("Alice", 30, Address("123 Main", "Springfield"))

attrs.asdict(p)
# {'name': 'Alice', 'age': 30, 'address': {'street': '123 Main', 'city': 'Springfield'}}

attrs.astuple(p)
# ('Alice', 30, ('123 Main', 'Springfield'))
```

**Filtering fields:**

```python
import attrs

# Exclude specific fields:
attrs.asdict(p, filter=attrs.filters.exclude(attrs.fields(Person).age))
# {'name': 'Alice', 'address': {'street': '123 Main', 'city': 'Springfield'}}

# Include only certain fields:
attrs.asdict(p, filter=attrs.filters.include(attrs.fields(Person).name))
# {'name': 'Alice'}

# Custom filter -- exclude private fields:
attrs.asdict(p, filter=lambda attr, value: not attr.name.startswith("_"))
```

**Custom value serializer:**

```python
import attrs
from datetime import datetime

@attrs.define
class Event:
    name: str
    timestamp: datetime

def serialize_value(inst, field, value):
    if isinstance(value, datetime):
        return value.isoformat()
    return value

e = Event("deploy", datetime(2025, 6, 15, 12, 0))
attrs.asdict(e, value_serializer=serialize_value)
# {'name': 'deploy', 'timestamp': '2025-06-15T12:00:00'}
```

### `attrs.evolve(inst, **changes)` -- Creating Modified Copies

Creates a new instance of the same class, with specified fields changed. This is the primary way to "update" frozen instances.

```python
import attrs

@attrs.frozen
class Config:
    host: str = "localhost"
    port: int = 8080
    debug: bool = False

config = Config()
dev_config = attrs.evolve(config, debug=True, port=3000)

print(config)      # Config(host='localhost', port=8080, debug=False)
print(dev_config)  # Config(host='localhost', port=3000, debug=True)

# Original is unchanged (frozen):
config.debug  # False
```

**Key behavior:** `evolve()` creates the new instance by calling `__init__`, which means validators and converters run on the changed values. This guarantees the new instance is also valid.

```python
import attrs

@attrs.frozen
class Score:
    value: int = attrs.field(validator=attrs.validators.and_(
        attrs.validators.instance_of(int),
        attrs.validators.ge(0),
        attrs.validators.le(100),
    ))

s = Score(85)
attrs.evolve(s, value=95)   # Score(value=95) -- valid
attrs.evolve(s, value=150)  # raises ValueError -- validator catches it
```

### `attrs.has(cls)` -- Checking if a Class Uses attrs

Returns `True` if the class is an attrs class.

```python
import attrs

@attrs.define
class MyClass:
    x: int

class PlainClass:
    pass

attrs.has(MyClass)    # True
attrs.has(PlainClass) # False
attrs.has(str)        # False
```

### `attrs.resolve_types(cls)` -- Resolving Forward References

Resolves string annotations (forward references) on an attrs class. This is necessary when you use `from __future__ import annotations` or string-based type hints and want attrs to know the actual types at runtime.

```python
from __future__ import annotations
import attrs

@attrs.define
class TreeNode:
    value: int
    children: list[TreeNode] = attrs.Factory(list)

# Without resolve_types, TreeNode's type annotations are strings.
# Calling resolve_types resolves them to actual types:
attrs.resolve_types(TreeNode)

# Now attrs.fields(TreeNode).children.type is list[TreeNode], not 'list[TreeNode]'
```

**When you need this:**
- When using `from __future__ import annotations` (PEP 563).
- When type hints contain forward references (e.g., self-referential classes).
- When validators or converters need to inspect `.type` at runtime.

You can pass `globalns` and `localns` dicts for resolution context:

```python
attrs.resolve_types(TreeNode, globalns=globals(), localns=locals())
```

---

## Validators

Validators run when an attribute is set (during `__init__` and, for mutable classes, on subsequent attribute assignment when `on_setattr` is configured). They receive three arguments: the instance, the attribute, and the value.

### Built-in Validators

#### `attrs.validators.instance_of(type)` -- Type Checking

```python
import attrs

@attrs.define
class User:
    name: str = attrs.field(validator=attrs.validators.instance_of(str))
    age: int = attrs.field(validator=attrs.validators.instance_of(int))

User("Alice", 30)     # OK
User("Alice", "old")  # raises TypeError: ("'age' must be <class 'int'>...")

# Multiple types:
@attrs.define
class Value:
    data: int | float = attrs.field(validator=attrs.validators.instance_of((int, float)))
```

#### `attrs.validators.in_(options)` -- Membership Check

```python
import attrs

@attrs.define
class Shirt:
    size: str = attrs.field(validator=attrs.validators.in_(["S", "M", "L", "XL"]))
    color: str = attrs.field(validator=attrs.validators.in_({"red", "blue", "green"}))

Shirt("M", "red")      # OK
Shirt("XXL", "red")    # raises ValueError: ("'size' must be in ['S', 'M', 'L', 'XL']...")
```

#### `attrs.validators.matches_re(pattern)` -- Regex Matching

```python
import attrs

@attrs.define
class Email:
    address: str = attrs.field(
        validator=attrs.validators.matches_re(r"^[^@]+@[^@]+\.[^@]+$")
    )

Email("user@example.com")  # OK
Email("invalid")            # raises ValueError
```

The `func` parameter controls which `re` method to use:

```python
# Use re.fullmatch (default is re.search):
attrs.validators.matches_re(r"^\d{5}$", func=re.fullmatch)
```

#### `attrs.validators.lt()`, `gt()`, `le()`, `ge()` -- Numeric Bounds

```python
import attrs

@attrs.define
class Temperature:
    celsius: float = attrs.field(validator=[
        attrs.validators.ge(-273.15),  # >= absolute zero
        attrs.validators.lt(1000),     # < 1000
    ])

@attrs.define
class Percentage:
    value: float = attrs.field(validator=[
        attrs.validators.ge(0),    # >= 0
        attrs.validators.le(100),  # <= 100
    ])
```

#### `attrs.validators.min_len()`, `max_len()` -- Length Validation

```python
import attrs

@attrs.define
class Password:
    value: str = attrs.field(validator=[
        attrs.validators.min_len(8),
        attrs.validators.max_len(128),
    ])

Password("short")              # raises ValueError (too short)
Password("validpassword123")   # OK
```

#### `attrs.validators.and_()` -- Composing Validators (All Must Pass)

Combines multiple validators; all must pass. Equivalent to passing a list, but more explicit.

```python
import attrs

positive_int = attrs.validators.and_(
    attrs.validators.instance_of(int),
    attrs.validators.gt(0),
)

@attrs.define
class Order:
    quantity: int = attrs.field(validator=positive_int)
```

**Note:** Passing a list of validators to `validator=[v1, v2, v3]` is implicitly `and_()`. The explicit form is useful when building reusable composite validators.

#### `attrs.validators.or_()` -- Composing Validators (Any Must Pass)

At least one validator must pass.

```python
import attrs

string_or_int = attrs.validators.or_(
    attrs.validators.instance_of(str),
    attrs.validators.instance_of(int),
)

@attrs.define
class Identifier:
    value: str | int = attrs.field(validator=string_or_int)

Identifier("abc")   # OK
Identifier(42)      # OK
Identifier(3.14)    # raises -- neither str nor int
```

#### `attrs.validators.optional()` -- Allow None

Wraps another validator to also accept `None`.

```python
import attrs

@attrs.define
class Profile:
    name: str = attrs.field(validator=attrs.validators.instance_of(str))
    bio: str | None = attrs.field(
        default=None,
        validator=attrs.validators.optional(attrs.validators.instance_of(str)),
    )

Profile("Alice")                # OK, bio=None
Profile("Alice", bio="Hello")   # OK
Profile("Alice", bio=42)        # raises TypeError
```

#### `attrs.validators.deep_iterable()` -- Validate Items in a Collection

Validates each element of an iterable, and optionally the iterable itself.

```python
import attrs

@attrs.define
class NumberList:
    values: list[int] = attrs.field(
        factory=list,
        validator=attrs.validators.deep_iterable(
            member_validator=attrs.validators.instance_of(int),
            iterable_validator=attrs.validators.instance_of(list),
        ),
    )

NumberList([1, 2, 3])      # OK
NumberList([1, "two", 3])  # raises TypeError on "two"
NumberList((1, 2, 3))      # raises TypeError -- not a list
```

#### `attrs.validators.deep_mapping()` -- Validate Keys and Values in a Dict

```python
import attrs

@attrs.define
class Headers:
    data: dict[str, str] = attrs.field(
        factory=dict,
        validator=attrs.validators.deep_mapping(
            key_validator=attrs.validators.instance_of(str),
            value_validator=attrs.validators.instance_of(str),
            mapping_validator=attrs.validators.instance_of(dict),
        ),
    )

Headers({"Content-Type": "text/html"})  # OK
Headers({1: "text/html"})               # raises TypeError on key
```

### Custom Validators

There are two ways to write custom validators.

#### Method 1: The `@field.validator` Decorator (Field-level)

Use this for validators tied to a specific field on a specific class. The decorated method receives the instance, attribute, and value.

```python
import attrs

@attrs.define
class Interval:
    lo: float
    hi: float

    @hi.validator
    def _check_hi(self, attribute, value):
        if value <= self.lo:
            raise ValueError(f"'hi' ({value}) must be greater than 'lo' ({self.lo})")

Interval(1.0, 5.0)   # OK
Interval(5.0, 1.0)   # raises ValueError
```

#### Method 2: Standalone Callable Validator

Any callable with the signature `(instance, attribute, value)` works as a validator. This is ideal for reusable validators shared across classes.

```python
import attrs

def validate_non_empty_string(instance, attribute, value):
    if not isinstance(value, str) or not value.strip():
        raise ValueError(f"'{attribute.name}' must be a non-empty string, got {value!r}")

@attrs.define
class Document:
    title: str = attrs.field(validator=validate_non_empty_string)
    body: str = attrs.field(validator=validate_non_empty_string)

Document("Hello", "World")  # OK
Document("", "World")       # raises ValueError
```

#### Method 3: Using `@attrs.validators.set_disabled()` / `@attrs.validators.disabled()`

You can temporarily disable all validators globally, which is useful for deserialization or testing:

```python
import attrs

with attrs.validators.disabled():
    # Validators do not run inside this block:
    item = Product(name="", price=-5)  # would normally fail validation
```

---

## Converters

Converters automatically transform input values during `__init__` (and on attribute assignment when `on_setattr` includes `convert`). They receive the value and return the converted value.

```python
import attrs

@attrs.define
class Config:
    port: int = attrs.field(converter=int)          # str -> int
    debug: bool = attrs.field(converter=bool)        # truthy/falsy -> bool
    tags: tuple = attrs.field(converter=tuple)        # any iterable -> tuple
    host: str = attrs.field(converter=str.lower)      # normalize to lowercase

c = Config(port="8080", debug=1, tags=["a", "b"], host="EXAMPLE.COM")
# Config(port=8080, debug=True, tags=('a', 'b'), host='example.com')
```

### Converters with Type Information

Since attrs 22.2.0, converters can optionally accept the value and the attribute:

```python
import attrs

def coerce_to_type(value, attribute=None):
    """Convert value to the field's annotated type."""
    if attribute and attribute.type:
        return attribute.type(value)
    return value
```

### Converters and Validators Interaction

Converters run **before** validators. This means validators see the already-converted value:

```python
import attrs

@attrs.define
class Port:
    number: int = attrs.field(
        converter=int,                          # "8080" -> 8080
        validator=attrs.validators.and_(
            attrs.validators.instance_of(int),  # sees 8080 (int), not "8080" (str)
            attrs.validators.ge(1),
            attrs.validators.le(65535),
        ),
    )

Port("8080")   # OK -- converter runs first, then validator sees 8080
Port("0")      # raises ValueError from ge(1) validator
```

### Optional Converter Pattern

A common pattern for fields that accept `None`:

```python
import attrs

def optional_int(value):
    """Convert to int, but pass through None."""
    if value is None:
        return None
    return int(value)

@attrs.define
class Query:
    limit: int | None = attrs.field(default=None, converter=optional_int)

Query(limit="50")   # Query(limit=50)
Query(limit=None)   # Query(limit=None)
Query()             # Query(limit=None)
```

Alternatively, use `attrs.converters.optional()` (available since attrs 17.1.0):

```python
import attrs

@attrs.define
class Query:
    limit: int | None = attrs.field(
        default=None,
        converter=attrs.converters.optional(int),
    )
```

### Pipe Converter

Chain multiple converters with `attrs.converters.pipe()`:

```python
import attrs

@attrs.define
class Tag:
    value: str = attrs.field(
        converter=attrs.converters.pipe(str, str.strip, str.lower)
    )

Tag("  HELLO  ")  # Tag(value='hello')
```

### `attrs.converters.to_bool()`

Converts common truthy/falsy string representations to `bool`:

```python
import attrs

@attrs.define
class Feature:
    enabled: bool = attrs.field(converter=attrs.converters.to_bool)

Feature("yes")   # Feature(enabled=True)
Feature("no")    # Feature(enabled=False)
Feature("true")  # Feature(enabled=True)
Feature("0")     # Feature(enabled=False)
Feature(1)       # Feature(enabled=True)
```

---

## Slots vs Dict Classes

### Slots Classes (`slots=True`, the default in modern API)

With `@attrs.define`, classes use `__slots__` by default. This means:

**Advantages:**
- **Lower memory usage** -- no per-instance `__dict__`.
- **Faster attribute access** -- slot access is implemented as a C-level descriptor.
- **No accidental attribute creation** -- assigning a typo like `inst.naem = "..."` raises `AttributeError`.
- **Plays well with `__weakref__`** -- weakref slot is added automatically.

**Limitations:**
- **No dynamic attributes** -- you cannot add attributes not declared in `__slots__`.
- **Multiple inheritance constraints** -- only one class in the MRO can have non-empty `__slots__` (unless carefully designed).
- **Cannot use `__dict__`-based introspection** -- `vars(instance)` raises `TypeError` (use `attrs.asdict()` instead).

```python
import attrs

@attrs.define  # slots=True by default
class SlottedUser:
    name: str
    age: int

u = SlottedUser("Alice", 30)
u.email = "a@b.com"  # AttributeError: 'SlottedUser' object has no attribute 'email'
```

### Dict Classes (`slots=False`)

```python
import attrs

@attrs.define(slots=False)
class DictUser:
    name: str
    age: int

u = DictUser("Alice", 30)
u.email = "a@b.com"  # Works -- dynamic attribute
vars(u)               # {'name': 'Alice', 'age': 30, 'email': 'a@b.com'}
```

**When to use `slots=False`:**
- You need dynamic attributes.
- You are mixing with third-party code that uses `__dict__` (e.g., some ORMs, pickle in older Python).
- You have complex multiple inheritance with other non-slotted classes.

**Recommendation:** Use slots (the default) unless you have a specific reason not to.

### Memory Comparison

```python
import sys, attrs

@attrs.define
class Slotted:
    x: int
    y: int
    z: int

@attrs.define(slots=False)
class DictBased:
    x: int
    y: int
    z: int

s = Slotted(1, 2, 3)
d = DictBased(1, 2, 3)

sys.getsizeof(s)  # ~56 bytes (approximate, no __dict__)
sys.getsizeof(d)  # ~48 bytes (but __dict__ adds ~104 bytes separately)
# Total for d: ~152 bytes. Slotted is significantly smaller.
```

---

## Type Annotations

attrs uses type annotations to discover fields. In the modern API (`@attrs.define`), any class-level annotation without a `ClassVar` or `InitVar` annotation becomes a field.

```python
import attrs
from typing import ClassVar

@attrs.define
class MyClass:
    x: int                    # field
    y: str = "hello"          # field with default
    z: list = attrs.Factory(list)  # field with factory default
    NOT_A_FIELD: ClassVar[int] = 42  # class variable, NOT a field
```

### Interaction with mypy and pyright

attrs ships with a **mypy plugin** and **PEP 681 `__dataclass_transform__`** support (Python 3.11+):

**mypy setup** -- add to `mypy.ini` or `pyproject.toml`:

```ini
# mypy.ini
[mypy]
plugins = attrs.mypy
```

```toml
# pyproject.toml
[tool.mypy]
plugins = ["attrs.mypy"]
```

This plugin teaches mypy about:
- Generated `__init__` signatures.
- Frozen instance immutability.
- Field types and defaults.
- `attrs.evolve()` type safety.

**pyright / pylance** -- starting with attrs 22.2.0 and pyright 1.1.234, attrs supports `__dataclass_transform__` (PEP 681), giving pyright native understanding of attrs classes without a plugin.

```python
# pyright understands this natively:
import attrs

@attrs.define
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x  # pyright knows this is float
```

### `from __future__ import annotations` Caveat

When using PEP 563 postponed annotations, all annotations become strings at runtime. attrs handles this for `__init__` generation, but if you need runtime type access (for validators, converters, or serialization), call `attrs.resolve_types()`:

```python
from __future__ import annotations
import attrs

@attrs.define
class Tree:
    value: int
    children: list[Tree] = attrs.Factory(list)

# Resolve string annotations to actual types:
attrs.resolve_types(Tree)
```

---

## on_setattr -- Controlling Attribute Setting

`on_setattr` controls what happens when you assign to an attribute after `__init__`. By default, `@attrs.define` uses `pipe(convert, validate)`, meaning converters and validators run on every assignment.

### Available Setattr Hooks

| Hook | Description |
|---|---|
| `attrs.setters.validate` | Run the field's validator on assignment |
| `attrs.setters.convert` | Run the field's converter on assignment |
| `attrs.setters.NO_OP` | Allow plain assignment with no checks |
| `attrs.setters.frozen` | Raise `FrozenInstanceError` on any assignment |
| `attrs.setters.pipe(*setters)` | Chain multiple setattr hooks together |

### Class-level on_setattr

```python
import attrs

# No validation on assignment (only on __init__):
@attrs.define(on_setattr=attrs.setters.NO_OP)
class FastMutable:
    x: int = attrs.field(validator=attrs.validators.instance_of(int))

f = FastMutable(42)      # validator runs during __init__
f.x = "not an int"       # NO validator runs -- this is allowed!
```

### Per-field on_setattr

```python
import attrs

@attrs.define
class Mixed:
    # This field validates on every set:
    name: str = attrs.field(validator=attrs.validators.instance_of(str))

    # This field only validates during __init__:
    cache: dict = attrs.field(
        factory=dict,
        on_setattr=attrs.setters.NO_OP,
    )
```

### Frozen Classes and on_setattr

`@attrs.frozen` sets `on_setattr=attrs.setters.frozen` on all fields, which raises `FrozenInstanceError` on any assignment attempt.

---

## Aliases

Aliases let you decouple the `__init__` parameter name from the attribute name. This is useful for fields with leading underscores (private attributes) or when you want a different public API.

```python
import attrs

@attrs.define
class Service:
    _url: str = attrs.field(alias="url")
    _timeout: int = attrs.field(alias="timeout", default=30)

# __init__ uses the alias names:
s = Service(url="https://api.example.com", timeout=60)
s._url      # 'https://api.example.com'
s._timeout  # 60
```

**Default alias behavior:** By default, attrs strips leading underscores from attribute names for `__init__` parameters. So `_url` automatically becomes `url` in `__init__` even without an explicit alias when using `@attrs.define`.

```python
import attrs

@attrs.define
class Config:
    _host: str       # __init__ parameter is 'host' (underscore stripped)
    _port: int = 80  # __init__ parameter is 'port'

c = Config(host="localhost", port=8080)
c._host  # 'localhost'
```

You can override this with an explicit `alias`:

```python
import attrs

@attrs.define
class Weird:
    _value: int = attrs.field(alias="val")

w = Weird(val=42)
w._value  # 42
```

---

## Comparison and Ordering

### Equality (`eq`)

By default, attrs generates `__eq__` and `__ne__` based on all fields.

```python
import attrs

@attrs.define
class Point:
    x: float
    y: float

Point(1, 2) == Point(1, 2)  # True
Point(1, 2) == Point(1, 3)  # False
```

**Exclude fields from equality:**

```python
import attrs

@attrs.define
class CachedResult:
    value: int
    cache_key: str = attrs.field(eq=False)  # ignored in comparisons

CachedResult(42, "key1") == CachedResult(42, "key2")  # True
```

**Custom equality key:**

```python
import attrs

@attrs.define
class CaseInsensitive:
    name: str = attrs.field(eq=str.lower)  # compare as lowercase

CaseInsensitive("Alice") == CaseInsensitive("alice")  # True
```

### Ordering (`order`)

When `order=True`, attrs generates `__lt__`, `__le__`, `__gt__`, `__ge__` using lexicographic comparison of fields.

```python
import attrs

@attrs.define(order=True)
class Version:
    major: int
    minor: int
    patch: int

Version(2, 0, 0) > Version(1, 9, 9)   # True
Version(1, 2, 3) < Version(1, 2, 4)   # True
sorted([Version(2,0,0), Version(1,0,0), Version(1,5,0)])
# [Version(1,0,0), Version(1,5,0), Version(2,0,0)]
```

### Hash (`hash`)

Hash behavior depends on `eq` and `frozen`:

| `eq` | `frozen` | `hash` behavior |
|---|---|---|
| `True` | `True` | `__hash__` generated (hashable) |
| `True` | `False` | `__hash__` set to `None` (unhashable) -- mutable + equality is unsafe to hash |
| `False` | `*` | `__hash__` from parent class (default `object.__hash__`) |

**Frozen classes are hashable by default:**

```python
import attrs

@attrs.frozen
class Color:
    r: int
    g: int
    b: int

colors = {Color(255, 0, 0), Color(0, 255, 0)}  # works in a set
lookup = {Color(255, 0, 0): "red"}               # works as dict key
```

**Force hashability on mutable classes** (use with caution -- only if you guarantee the hashed fields never change):

```python
import attrs

@attrs.define(hash=True)  # explicitly enable -- you promise not to mutate
class Tag:
    name: str
```

---

## attrs vs dataclasses

`attrs` predates `dataclasses` (stdlib since Python 3.7) and directly inspired it. Here is a feature comparison:

| Feature | `attrs` | `dataclasses` |
|---|---|---|
| **Validators** | Built-in, composable, rich library | No built-in support |
| **Converters** | Built-in, run on init and optionally on setattr | No built-in support |
| **Slots by default** | Yes (`@attrs.define`) | No (opt-in via `slots=True` in 3.10+) |
| **Frozen classes** | `@attrs.frozen` | `@dataclass(frozen=True)` |
| **`evolve()`** | `attrs.evolve()` | `dataclasses.replace()` (equivalent) |
| **`asdict()` / `astuple()`** | Yes | Yes |
| **`on_setattr` hooks** | Yes -- validate, convert, pipe | No |
| **Aliases** | Yes | No (added in 3.13 as `alias` on `field()`) |
| **`Factory(takes_self=True)`** | Yes | No (only `default_factory`, no self access) |
| **Custom `__repr__`** | Per-field `repr` callables | Only `repr=True/False` per field |
| **Custom `eq` key** | `eq=str.lower` etc. | No |
| **Metadata** | Yes | Yes |
| **`__match_args__`** | Yes | Yes (3.10+) |
| **mypy/pyright support** | Plugin + PEP 681 | Native |
| **No third-party dependency** | Requires `pip install attrs` | Stdlib, zero install |
| **Performance** | Essentially identical | Essentially identical |

**When to use attrs:**
- You need validators or converters.
- You need `on_setattr` hooks.
- You want slots by default without remembering to set `slots=True`.
- You need `Factory(takes_self=True)` for self-referential defaults.
- You need field aliases.
- You are building a library that needs to support Python < 3.10 with slots.
- **You are working in our stack** -- Rune, Transmute, and Enchant all use attrs.

**When to use dataclasses:**
- You want zero third-party dependencies.
- Your classes are simple data holders with no validation needs.
- Your team is more familiar with stdlib patterns.

---

## Gotchas and Common Mistakes

### 1. Mutable Default Gotcha -- Always Use Factory

Never pass a mutable object (list, dict, set) as a `default`. This is the same class of bug as the classic Python mutable default argument. attrs will **raise an error** if you try, but it is important to understand why.

```python
import attrs

# BAD -- attrs raises ValueError to protect you:
@attrs.define
class BadTeam:
    members: list = []  # ValueError! attrs prevents shared mutable defaults

# GOOD -- use Factory:
@attrs.define
class GoodTeam:
    members: list = attrs.Factory(list)   # new list per instance

# ALSO GOOD -- shorthand:
@attrs.define
class AlsoGoodTeam:
    members: list = attrs.field(factory=list)
```

### 2. Slots and Inheritance Limitations

When using slotted classes (the default), multiple inheritance requires care. Only one class in the MRO should define `__slots__` (unless the others have empty `__slots__`).

```python
import attrs

@attrs.define
class Base:
    x: int

@attrs.define
class Child(Base):
    y: int   # OK -- attrs handles single inheritance correctly

# PROBLEMATIC -- two slotted classes with fields in MRO:
@attrs.define
class Mixin:
    z: int

# This may cause issues with diamond inheritance or conflicting slots.
# Solution: use slots=False on mixins, or design your hierarchy carefully.
@attrs.define(slots=False)
class SafeMixin:
    z: int = 0
```

### 3. `evolve()` Runs Validators

`attrs.evolve()` creates a new instance via `__init__`, so all validators and converters run. This is usually desirable but can be surprising if you expect a raw copy.

```python
import attrs

@attrs.frozen
class Range:
    lo: int
    hi: int

    @hi.validator
    def _check_hi(self, attribute, value):
        if value < self.lo:
            raise ValueError(f"hi ({value}) must be >= lo ({self.lo})")

r = Range(0, 10)
attrs.evolve(r, hi=5)    # OK
attrs.evolve(r, hi=-1)   # raises ValueError -- validator still runs
attrs.evolve(r, lo=20)   # raises ValueError -- hi=10 < lo=20
# To change both, pass both: attrs.evolve(r, lo=20, hi=30)
```

### 4. Forward Reference Resolution

If you use `from __future__ import annotations` or string-based type hints, call `attrs.resolve_types()` when you need runtime type information.

```python
from __future__ import annotations
import attrs

@attrs.define
class Node:
    children: list[Node] = attrs.Factory(list)

# If you use cattrs or inspect .type at runtime, you MUST call:
attrs.resolve_types(Node)
```

Without `resolve_types()`, `attrs.fields(Node).children.type` will be the string `'list[Node]'` instead of the actual type.

### 5. Performance of Validators on Hot Paths

Validators add overhead to every instance creation and (by default) every attribute assignment. On hot paths where you create millions of objects:

```python
import attrs

# If you create millions of these, validators add up:
@attrs.define
class HotPathPoint:
    x: float = attrs.field(validator=attrs.validators.instance_of(float))
    y: float = attrs.field(validator=attrs.validators.instance_of(float))

# Options for performance-critical code:
# 1. Disable on_setattr (only validate during __init__):
@attrs.define(on_setattr=attrs.setters.NO_OP)
class FasterPoint:
    x: float = attrs.field(validator=attrs.validators.instance_of(float))
    y: float

# 2. Skip validators entirely when you trust the data:
with attrs.validators.disabled():
    points = [HotPathPoint(x, y) for x, y in huge_dataset]

# 3. Use no validators at all:
@attrs.define
class FastestPoint:
    x: float
    y: float
```

### 6. `__attrs_post_init__` for Complex Initialization

When you need custom initialization logic that runs after attrs generates `__init__`, define `__attrs_post_init__`. This is the place for derived attributes, validation involving multiple fields, or registering the instance somewhere.

```python
import attrs
import math

@attrs.define
class Circle:
    radius: float = attrs.field(validator=attrs.validators.gt(0))
    _area: float = attrs.field(init=False)
    _circumference: float = attrs.field(init=False)

    def __attrs_post_init__(self):
        self._area = math.pi * self.radius ** 2
        self._circumference = 2 * math.pi * self.radius

c = Circle(5.0)
c._area           # 78.5398...
c._circumference  # 31.4159...
```

**Important:** For `@attrs.frozen` classes, you must use `object.__setattr__` in `__attrs_post_init__` because the instance is already frozen:

```python
import attrs

@attrs.frozen
class FrozenCircle:
    radius: float
    _area: float = attrs.field(init=False)

    def __attrs_post_init__(self):
        # Cannot do self._area = ... (frozen!)
        object.__setattr__(self, "_area", 3.14159 * self.radius ** 2)
```

### 7. Forgetting `eq=False` on Non-Value Fields

Fields that represent caches, loggers, or connections should not participate in equality or hashing:

```python
import attrs
import logging

@attrs.define
class Service:
    name: str
    url: str
    _logger: logging.Logger = attrs.field(
        init=False,
        repr=False,
        eq=False,
        hash=False,
    )

    def __attrs_post_init__(self):
        self._logger = logging.getLogger(self.name)

# Two services with the same name/url are equal,
# regardless of logger identity:
Service("api", "http://a") == Service("api", "http://a")  # True
```

---

## Complete Code Examples

### Example 1: Basic `@attrs.define` and `@attrs.frozen`

```python
import attrs

# Mutable class with slots (default):
@attrs.define
class MutablePoint:
    x: float
    y: float

p = MutablePoint(1.0, 2.0)
p.x = 3.0         # allowed
print(p)           # MutablePoint(x=3.0, y=2.0)


# Immutable (frozen) class:
@attrs.frozen
class FrozenPoint:
    x: float
    y: float

fp = FrozenPoint(1.0, 2.0)
# fp.x = 3.0      # raises FrozenInstanceError
print(fp)          # FrozenPoint(x=1.0, y=2.0)

# Frozen instances are hashable:
point_set = {FrozenPoint(0, 0), FrozenPoint(1, 1), FrozenPoint(0, 0)}
print(len(point_set))  # 2 (duplicates removed)
```

### Example 2: Validators and Converters

```python
import attrs
from datetime import datetime

def parse_datetime(value):
    """Convert string or datetime to datetime."""
    if isinstance(value, datetime):
        return value
    return datetime.fromisoformat(value)

@attrs.define
class Event:
    name: str = attrs.field(
        validator=[
            attrs.validators.instance_of(str),
            attrs.validators.min_len(1),
        ]
    )
    priority: int = attrs.field(
        converter=int,
        validator=[
            attrs.validators.ge(1),
            attrs.validators.le(5),
        ],
    )
    timestamp: datetime = attrs.field(converter=parse_datetime)
    tags: frozenset = attrs.field(
        converter=frozenset,
        factory=frozenset,
    )

# String priority is converted to int, then validated:
e = Event(
    name="deployment",
    priority="3",
    timestamp="2025-06-15T10:30:00",
    tags=["prod", "release"],
)
print(e)
# Event(name='deployment', priority=3, timestamp=datetime(2025, 6, 15, 10, 30), tags=frozenset({'prod', 'release'}))

# Validation failure:
try:
    Event(name="x", priority="10", timestamp="2025-01-01")
except ValueError as exc:
    print(exc)  # priority must be <= 5
```

### Example 3: Factory Defaults

```python
import attrs
import uuid
import time

@attrs.define
class Task:
    title: str
    id: str = attrs.Factory(lambda: uuid.uuid4().hex[:8])
    created_at: float = attrs.Factory(time.time)
    subtasks: list = attrs.Factory(list)
    metadata: dict = attrs.Factory(dict)

t1 = Task("Write docs")
t2 = Task("Review PR")

# Each instance gets its own id, timestamp, list, and dict:
print(t1.id != t2.id)              # True
print(t1.subtasks is not t2.subtasks)  # True


# Factory with takes_self:
@attrs.define
class Config:
    env: str = "production"
    debug: bool = attrs.Factory(
        lambda self: self.env != "production",
        takes_self=True,
    )
    log_level: str = attrs.Factory(
        lambda self: "DEBUG" if self.debug else "WARNING",
        takes_self=True,
    )

print(Config())                    # Config(env='production', debug=False, log_level='WARNING')
print(Config(env="development"))   # Config(env='development', debug=True, log_level='DEBUG')
```

### Example 4: `evolve()` for Immutable Updates

```python
import attrs

@attrs.frozen
class AppConfig:
    host: str = "0.0.0.0"
    port: int = 8080
    workers: int = 4
    debug: bool = False
    db_url: str = "sqlite:///app.db"

# Start with production defaults:
prod = AppConfig()

# Create development config by evolving:
dev = attrs.evolve(prod, debug=True, port=3000, db_url="sqlite:///dev.db")

# Create staging config from production:
staging = attrs.evolve(prod, workers=2, db_url="postgresql://staging/app")

print(prod)
# AppConfig(host='0.0.0.0', port=8080, workers=4, debug=False, db_url='sqlite:///app.db')

print(dev)
# AppConfig(host='0.0.0.0', port=3000, workers=4, debug=True, db_url='sqlite:///dev.db')

print(staging)
# AppConfig(host='0.0.0.0', port=8080, workers=2, debug=False, db_url='postgresql://staging/app')
```

### Example 5: Nested attrs Classes

```python
import attrs

@attrs.frozen
class Address:
    street: str
    city: str
    state: str
    zip_code: str

@attrs.frozen
class Company:
    name: str
    industry: str

@attrs.frozen
class Employee:
    name: str
    email: str
    company: Company
    address: Address
    salary: float = attrs.field(repr=False)  # hide from repr

emp = Employee(
    name="Alice Johnson",
    email="alice@example.com",
    company=Company("Acme Corp", "Technology"),
    address=Address("123 Main St", "Springfield", "IL", "62704"),
    salary=95000.0,
)

# Recursive asdict:
d = attrs.asdict(emp)
print(d)
# {
#     'name': 'Alice Johnson',
#     'email': 'alice@example.com',
#     'company': {'name': 'Acme Corp', 'industry': 'Technology'},
#     'address': {'street': '123 Main St', 'city': 'Springfield',
#                 'state': 'IL', 'zip_code': '62704'},
#     'salary': 95000.0,
# }

# Update nested value using evolve:
new_address = attrs.evolve(emp.address, city="Chicago", zip_code="60601")
moved_emp = attrs.evolve(emp, address=new_address)
print(moved_emp.address)
# Address(street='123 Main St', city='Chicago', state='IL', zip_code='60601')
```

### Example 6: Custom Validators with Cross-Field Checks

```python
import attrs
from datetime import date

@attrs.define
class DateRange:
    start: date = attrs.field(converter=lambda v: v if isinstance(v, date) else date.fromisoformat(v))
    end: date = attrs.field(converter=lambda v: v if isinstance(v, date) else date.fromisoformat(v))

    @end.validator
    def _validate_end(self, attribute, value):
        if value < self.start:
            raise ValueError(
                f"'end' ({value}) must be on or after 'start' ({self.start})"
            )

    @property
    def days(self) -> int:
        return (self.end - self.start).days


@attrs.define
class Booking:
    guest_name: str = attrs.field(
        validator=[
            attrs.validators.instance_of(str),
            attrs.validators.min_len(1),
        ]
    )
    room: int = attrs.field(
        validator=[
            attrs.validators.instance_of(int),
            attrs.validators.ge(100),
            attrs.validators.lt(1000),
        ]
    )
    dates: DateRange
    guests: int = attrs.field(
        default=1,
        validator=[
            attrs.validators.instance_of(int),
            attrs.validators.ge(1),
            attrs.validators.le(10),
        ]
    )

# Valid booking:
b = Booking(
    guest_name="Alice",
    room=205,
    dates=DateRange("2025-07-01", "2025-07-05"),
    guests=2,
)
print(f"Booking for {b.dates.days} days")  # Booking for 4 days

# Invalid -- end before start:
try:
    DateRange("2025-07-05", "2025-07-01")
except ValueError as e:
    print(e)  # 'end' (2025-07-01) must be on or after 'start' (2025-07-05)
```

### Example 7: Integration with cattrs for Serialization

[cattrs](https://catt.rs/) is the companion library for attrs that handles structuring (dict -> attrs) and unstructuring (attrs -> dict) with full type awareness.

```python
import attrs
import cattrs
from datetime import datetime

@attrs.frozen
class Address:
    street: str
    city: str
    zip_code: str

@attrs.frozen
class User:
    name: str
    email: str
    address: Address
    created_at: datetime
    tags: frozenset[str] = attrs.Factory(frozenset)

# --- Unstructuring (attrs -> dict) ---

user = User(
    name="Alice",
    email="alice@example.com",
    address=Address("123 Main", "Springfield", "62704"),
    created_at=datetime(2025, 6, 15, 12, 0),
    tags=frozenset(["admin", "verified"]),
)

converter = cattrs.Converter()

# Register a custom hook for datetime:
converter.register_unstructure_hook(
    datetime, lambda dt: dt.isoformat()
)
converter.register_structure_hook(
    datetime, lambda s, _: datetime.fromisoformat(s)
)

# Unstructure to dict (for JSON serialization):
data = converter.unstructure(user)
print(data)
# {
#     'name': 'Alice',
#     'email': 'alice@example.com',
#     'address': {'street': '123 Main', 'city': 'Springfield', 'zip_code': '62704'},
#     'created_at': '2025-06-15T12:00:00',
#     'tags': ['admin', 'verified'],
# }

# --- Structuring (dict -> attrs) ---

raw_data = {
    "name": "Bob",
    "email": "bob@test.com",
    "address": {"street": "456 Oak", "city": "Portland", "zip_code": "97201"},
    "created_at": "2025-06-20T08:30:00",
    "tags": ["new"],
}

bob = converter.structure(raw_data, User)
print(bob)
# User(name='Bob', email='bob@test.com', address=Address(...), ...)
print(type(bob))           # <class 'User'>
print(type(bob.address))   # <class 'Address'>
print(type(bob.created_at))  # <class 'datetime.datetime'>
```

**cattrs key patterns:**

```python
import cattrs

converter = cattrs.Converter()

# Structure a list of objects:
users = converter.structure(
    [{"name": "A", "email": "a@b.com", "address": {...}, "created_at": "..."},
     {"name": "B", "email": "b@c.com", "address": {...}, "created_at": "..."}],
    list[User],
)

# Handle optional fields:
@attrs.frozen
class Profile:
    name: str
    bio: str | None = None

profile = converter.structure({"name": "Alice"}, Profile)
# Profile(name='Alice', bio=None)
```

---

## Quick Reference

### Decorators

| Decorator | Description |
|---|---|
| `@attrs.define` | Modern mutable class (`slots=True`) |
| `@attrs.frozen` | Immutable class (`frozen=True`, `slots=True`, hashable) |
| `@attrs.mutable` | Alias for `@attrs.define` |

### Functions

| Function | Description |
|---|---|
| `attrs.field()` | Configure a field (default, validator, converter, alias, etc.) |
| `attrs.Factory(fn, takes_self=False)` | Lazy default factory |
| `attrs.fields(cls)` | Return tuple of `Attribute` objects for the class |
| `attrs.asdict(inst)` | Recursively convert instance to dict |
| `attrs.astuple(inst)` | Recursively convert instance to tuple |
| `attrs.evolve(inst, **changes)` | Create a new instance with changes applied |
| `attrs.has(cls)` | Return `True` if `cls` is an attrs class |
| `attrs.resolve_types(cls)` | Resolve string annotations to real types |
| `attrs.validators.disabled()` | Context manager to disable all validators |

### Validators

| Validator | Description |
|---|---|
| `instance_of(type)` | Check `isinstance` |
| `in_(options)` | Check membership in a collection |
| `matches_re(pattern)` | Regex match |
| `gt(val)` / `ge(val)` | Greater than / greater-or-equal |
| `lt(val)` / `le(val)` | Less than / less-or-equal |
| `min_len(n)` / `max_len(n)` | Minimum / maximum length |
| `and_(*validators)` | All must pass |
| `or_(*validators)` | At least one must pass |
| `optional(validator)` | Allow `None`, otherwise run validator |
| `deep_iterable(member, iterable)` | Validate iterable elements |
| `deep_mapping(key, value, mapping)` | Validate mapping keys and values |

### Converters

| Converter | Description |
|---|---|
| `attrs.converters.optional(converter)` | Pass through `None`, otherwise run converter |
| `attrs.converters.pipe(*converters)` | Chain multiple converters |
| `attrs.converters.to_bool` | Convert strings/ints to `bool` |

### Setattr Hooks

| Hook | Description |
|---|---|
| `attrs.setters.validate` | Run validators on attribute set |
| `attrs.setters.convert` | Run converters on attribute set |
| `attrs.setters.NO_OP` | Plain assignment, no hooks |
| `attrs.setters.frozen` | Forbid assignment (raise error) |
| `attrs.setters.pipe(*hooks)` | Chain multiple hooks |

---

## Further Reading

- [Official documentation](https://www.attrs.org/en/stable/)
- [GitHub repository](https://github.com/python-attrs/attrs)
- [PyPI page](https://pypi.org/project/attrs/)
- [cattrs documentation](https://catt.rs/en/stable/)
- [PEP 557 -- Data Classes](https://peps.python.org/pep-0557/) -- the PEP that attrs inspired
- [PEP 681 -- Data Class Transforms](https://peps.python.org/pep-0681/) -- enables pyright support
- [Why not just use dataclasses?](https://www.attrs.org/en/stable/why.html)
