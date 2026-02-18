# PyYAML — API Reference

> Part of the pyyaml skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Security Warning](#security-warning)
- [Core API Reference](#core-api-reference)
  - [Loading (Parsing) YAML](#loading-parsing-yaml)
  - [Loader Classes](#loader-classes)
  - [Dumping (Emitting) YAML](#dumping-emitting-yaml)
  - [Dumper Classes](#dumper-classes)
- [Custom Representers and Constructors](#custom-representers-and-constructors)

## SECURITY WARNING

**Never use `yaml.load()` or `yaml.dump()` with untrusted input without specifying a safe Loader/Dumper.**

`yaml.load()` with the default `Loader` (or `FullLoader`) can execute arbitrary Python code via YAML tags like `!!python/object/apply:os.system`. This is a well-known and actively exploited vulnerability.

```python
# DANGEROUS -- arbitrary code execution possible
data = yaml.load(untrusted_input)                    # DO NOT DO THIS
data = yaml.load(untrusted_input, Loader=yaml.Loader)  # ALSO DANGEROUS

# SAFE -- use safe_load or SafeLoader
data = yaml.safe_load(untrusted_input)               # CORRECT
data = yaml.load(untrusted_input, Loader=yaml.SafeLoader)  # ALSO CORRECT
```

Starting with PyYAML 5.1, calling `yaml.load()` without an explicit `Loader` raises a warning. Starting with PyYAML 6.0, it raises an error.

---

## Core API Reference

### Loading (Parsing) YAML

#### `yaml.safe_load(stream)` -- Recommended

Parses a single YAML document. Only constructs basic Python types (dict, list, str, int, float, bool, None, datetime).

```python
import yaml

data = yaml.safe_load("""
name: Alice
age: 30
languages:
  - Python
  - Rust
active: true
""")

# {'name': 'Alice', 'age': 30, 'languages': ['Python', 'Rust'], 'active': True}
```

#### `yaml.safe_load_all(stream)` -- Multiple Documents

Parses a YAML stream containing multiple documents separated by `---`.

```python
docs = list(yaml.safe_load_all("""
---
name: Alice
---
name: Bob
---
name: Charlie
"""))
# [{'name': 'Alice'}, {'name': 'Bob'}, {'name': 'Charlie'}]
```

#### `yaml.load(stream, Loader=...)` -- Full Control

Use when you need custom tags or Python object deserialization (only with trusted input).

```python
# Only for TRUSTED input
data = yaml.load(stream, Loader=yaml.FullLoader)
```

### Loader Classes

| Loader | Description | Safety |
|--------|-------------|--------|
| `BaseLoader` | Only basic YAML types, no type coercion | Safe |
| `SafeLoader` | Standard Python types (str, int, float, list, dict, etc.) | Safe |
| `FullLoader` | All YAML tags except arbitrary Python objects | Mostly safe |
| `UnsafeLoader` | All YAML tags including Python object instantiation | **DANGEROUS** |
| `Loader` | Alias for `UnsafeLoader` (historical) | **DANGEROUS** |
| `CBaseLoader` | C-accelerated `BaseLoader` | Safe |
| `CSafeLoader` | C-accelerated `SafeLoader` | Safe |
| `CFullLoader` | C-accelerated `FullLoader` | Mostly safe |
| `CUnsafeLoader` | C-accelerated `UnsafeLoader` | **DANGEROUS** |
| `CLoader` | C-accelerated `Loader` | **DANGEROUS** |

### Dumping (Emitting) YAML

#### `yaml.safe_dump(data, stream=None, **kwargs)` -- Recommended

Serializes Python objects to a YAML string (or writes to a stream).

```python
import yaml

data = {
    "name": "Alice",
    "scores": [95, 87, 92],
    "metadata": {"role": "admin", "active": True},
}

output = yaml.safe_dump(data)
print(output)
# metadata:
#   active: true
#   role: admin
# name: Alice
# scores:
# - 95
# - 87
# - 92
```

#### Key Parameters for `safe_dump()`

| Parameter | Default | Description |
|-----------|---------|-------------|
| `default_flow_style` | `False` | `True` for inline JSON-like style, `False` for block style |
| `default_style` | `None` | Quote style for scalars: `None`, `"'"`, `'"'`, `'\|'`, `'>'` |
| `sort_keys` | `True` | Sort dictionary keys alphabetically |
| `indent` | `2` | Indentation width |
| `width` | `80` | Line width before wrapping |
| `allow_unicode` | `True` | Allow unicode characters in output |
| `explicit_start` | `False` | Add `---` document start marker |
| `explicit_end` | `False` | Add `...` document end marker |

```python
# Unsorted keys, explicit document markers
yaml.safe_dump(data, sort_keys=False, explicit_start=True, explicit_end=True)
# ---
# name: Alice
# scores:
# - 95
# - 87
# - 92
# metadata:
#   role: admin
#   active: true
# ...

# Write to file
with open("config.yaml", "w") as f:
    yaml.safe_dump(data, f)
```

#### `yaml.safe_dump_all(documents, stream=None, **kwargs)`

Dumps multiple documents to a single YAML stream:

```python
docs = [{"name": "Alice"}, {"name": "Bob"}]
print(yaml.safe_dump_all(docs))
# name: Alice
# ---
# name: Bob
```

#### `yaml.dump(data, stream=None, Dumper=..., **kwargs)`

Full-control dumping, needed for custom representers:

```python
yaml.dump(data, Dumper=yaml.SafeDumper)
```

### Dumper Classes

| Dumper | Description |
|--------|-------------|
| `BaseDumper` | Minimal, only basic types |
| `SafeDumper` | Standard Python types (used by `safe_dump`) |
| `Dumper` | All Python types including objects |
| `CDumper` | C-accelerated `Dumper` |
| `CSafeDumper` | C-accelerated `SafeDumper` |

---

## Custom Representers and Constructors

### Custom Representer (Python -> YAML)

Teach PyYAML how to serialize custom types:

```python
import yaml
from datetime import date

class Person:
    def __init__(self, name, birth):
        self.name = name
        self.birth = birth

def person_representer(dumper, person):
    return dumper.represent_mapping("!person", {
        "name": person.name,
        "birth": person.birth,
    })

yaml.add_representer(Person, person_representer, Dumper=yaml.SafeDumper)

p = Person("Alice", date(1990, 5, 15))
print(yaml.safe_dump(p))
# !person
# birth: 1990-05-15
# name: Alice
```

### Custom Constructor (YAML -> Python)

Teach PyYAML how to deserialize custom tags:

```python
def person_constructor(loader, node):
    values = loader.construct_mapping(node)
    return Person(values["name"], values["birth"])

yaml.add_constructor("!person", person_constructor, Loader=yaml.SafeLoader)

data = yaml.safe_load("""
!person
name: Alice
birth: 1990-05-15
""")
# Person(name='Alice', birth=datetime.date(1990, 5, 15))
```
