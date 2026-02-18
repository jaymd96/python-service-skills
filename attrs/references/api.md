# attrs — API Reference

> Part of the attrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Overview and Philosophy](#overview-and-philosophy)
- [Core API](#core-api)
  - [@attrs.define -- Modern Mutable Classes](#attrsddefine----modern-mutable-classes)
  - [@attrs.frozen -- Immutable Classes](#attrsfrozen----immutable-classes)
  - [@attrs.mutable -- Alias for @attrs.define](#attrsmutable----alias-for-attrsdefine)
  - [attrs.field() -- Field Configuration](#attrsfield----field-configuration)
  - [attrs.Factory -- Lazy Defaults](#attrsfactoryfunc-takes_selffalse----lazy-defaults)
  - [attrs.NOTHING -- Sentinel Value](#attrsnothing----sentinel-value)
  - [attrs.fields(cls) -- Introspecting Fields](#attrsfieldscls----introspecting-fields)
  - [attrs.has(cls) -- Checking if a Class Uses attrs](#attrshascls----checking-if-a-class-uses-attrs)
  - [attrs.resolve_types(cls) -- Resolving Forward References](#attrsresolve_typescls----resolving-forward-references)
- [Type Annotations](#type-annotations)
- [Quick Reference](#quick-reference)

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
| `attrs.has(cls)` | Return `True` if `cls` is an attrs class |
| `attrs.resolve_types(cls)` | Resolve string annotations to real types |
