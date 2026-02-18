# attrs — Examples & Gotchas

> Part of the attrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Mutable Default Gotcha](#1-mutable-default-gotcha----always-use-factory)
  - [2. Slots and Inheritance Limitations](#2-slots-and-inheritance-limitations)
  - [3. evolve() Runs Validators](#3-evolve-runs-validators)
  - [4. Forward Reference Resolution](#4-forward-reference-resolution)
  - [5. Performance of Validators on Hot Paths](#5-performance-of-validators-on-hot-paths)
  - [6. __attrs_post_init__ for Complex Initialization](#6-__attrs_post_init__-for-complex-initialization)
  - [7. Forgetting eq=False on Non-Value Fields](#7-forgetting-eqfalse-on-non-value-fields)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic @attrs.define and @attrs.frozen](#example-1-basic-attrsdefine-and-attrsfrozen)
  - [Example 2: Validators and Converters](#example-2-validators-and-converters)
  - [Example 3: Factory Defaults](#example-3-factory-defaults)
  - [Example 4: evolve() for Immutable Updates](#example-4-evolve-for-immutable-updates)
  - [Example 5: Nested attrs Classes](#example-5-nested-attrs-classes)
  - [Example 6: Custom Validators with Cross-Field Checks](#example-6-custom-validators-with-cross-field-checks)
  - [Example 7: Integration with cattrs for Serialization](#example-7-integration-with-cattrs-for-serialization)
- [Further Reading](#further-reading)

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

## Further Reading

- [Official documentation](https://www.attrs.org/en/stable/)
- [GitHub repository](https://github.com/python-attrs/attrs)
- [PyPI page](https://pypi.org/project/attrs/)
- [cattrs documentation](https://catt.rs/en/stable/)
- [PEP 557 -- Data Classes](https://peps.python.org/pep-0557/) -- the PEP that attrs inspired
- [PEP 681 -- Data Class Transforms](https://peps.python.org/pep-0681/) -- enables pyright support
- [Why not just use dataclasses?](https://www.attrs.org/en/stable/why.html)
