---
name: cattrs
description: Composable converters for structuring and unstructuring data with cattrs. Use when converting between dicts and typed objects, serializing attrs classes, setting up custom structure/unstructure hooks, handling unions/tagged types, or configuring preconf converters for JSON/YAML/msgpack. Triggers on cattrs, structure, unstructure, converter hooks, dict to object, serialization with attrs.
---

# cattrs — Structured ↔ Unstructured conversion (v24.1.2)

## Quick Start

```bash
pip install cattrs
```

```python
import attrs, cattrs

@attrs.define
class User:
    name: str
    age: int

# dict → object
user = cattrs.structure({"name": "Alice", "age": 30}, User)

# object → dict
data = cattrs.unstructure(user)  # {"name": "Alice", "age": 30}
```

## Key Patterns

### Custom converter with hooks
```python
converter = cattrs.GenConverter()  # or cattrs.Converter()
converter.register_structure_hook(
    datetime, lambda v, _: datetime.fromisoformat(v)
)
converter.register_unstructure_hook(
    datetime, lambda v: v.isoformat()
)
```

### Tagged unions (polymorphism)
```python
from cattrs.strategies import configure_tagged_union
configure_tagged_union(Dog | Cat, converter, tag_name="type")
```

### Preconf converters (JSON-ready)
```python
from cattrs.preconf.orjson import make_converter
converter = make_converter()  # handles datetime, UUID, etc.
```

## References

- **[api.md](references/api.md)** — Core structure/unstructure API, Converter basics
- **[converters.md](references/converters.md)** — Custom hooks, GenConverter, strategies, preconf converters
- **[type-support.md](references/type-support.md)** — Supported types: attrs, dataclasses, TypedDict, Union, generics
- **[examples.md](references/examples.md)** — Complete examples, gotchas, migration patterns

## Grep Patterns

- `cattrs\.structure|cattrs\.unstructure` — Find structuring calls
- `register_structure_hook|register_unstructure_hook` — Find custom hooks
- `GenConverter|Converter\(` — Find converter setup
