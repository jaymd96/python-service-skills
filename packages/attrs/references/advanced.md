# attrs — Advanced Features

> Part of the attrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [attrs.asdict() and attrs.astuple() -- Serialization Helpers](#attrsasdict-and-attrsastuple----serialization-helpers)
- [attrs.evolve() -- Creating Modified Copies](#attrsevolveinst-changes----creating-modified-copies)
- [Slots vs Dict Classes](#slots-vs-dict-classes)
- [on_setattr -- Controlling Attribute Setting](#on_setattr----controlling-attribute-setting)
- [Aliases](#aliases)
- [Comparison and Ordering](#comparison-and-ordering)
- [__attrs_post_init__ for Complex Initialization](#__attrs_post_init__-for-complex-initialization)
- [attrs vs dataclasses](#attrs-vs-dataclasses)

## `attrs.asdict()` and `attrs.astuple()` -- Serialization Helpers

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

## `attrs.evolve(inst, **changes)` -- Creating Modified Copies

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

## `__attrs_post_init__` for Complex Initialization

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

### Setattr Hooks Quick Reference

| Hook | Description |
|---|---|
| `attrs.setters.validate` | Run validators on attribute set |
| `attrs.setters.convert` | Run converters on attribute set |
| `attrs.setters.NO_OP` | Plain assignment, no hooks |
| `attrs.setters.frozen` | Forbid assignment (raise error) |
| `attrs.setters.pipe(*hooks)` | Chain multiple hooks |
