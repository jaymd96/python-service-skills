# cattrs — Type Support

> Part of the cattrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

1. [attrs Classes](#attrs-classes)
2. [Dataclasses](#dataclasses)
3. [Optional and Union](#optionalt-and-uniont1-t2-)
4. [Literal](#literal)
5. [Collection Types](#collection-types-list-dict-set-tuple-frozenset)
6. [datetime, date, UUID, Path, Enum](#datetime-date-uuid-path-enum)
7. [TypedDict](#typingtypeddict)
8. [NamedTuple](#typingnamedtuple)
9. [Forward References](#forward-references)
10. [Generic Types](#generic-types-t-listt)

---

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

**Important**: For `Union[TypeA, TypeB]` (where neither is `None`), cattrs cannot automatically determine which type to use. You must register a custom hook or use `configure_tagged_union`. See [Gotchas](examples.md#2-union-disambiguation-cattrs-cannot-always-auto-detect).

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
