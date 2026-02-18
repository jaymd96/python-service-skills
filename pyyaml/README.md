# PyYAML

A full-featured **YAML parser and emitter** for Python.

`PyYAML` can parse YAML 1.1 documents into Python objects and serialize (dump) Python objects back to YAML. It supports all YAML types, custom tags, multiple documents in a single stream, and both pure-Python and C-accelerated (LibYAML) backends. PyYAML is one of the most widely used Python packages, depended upon by Ansible, Docker Compose, Kubernetes tools, and many configuration-driven projects.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `PyYAML` |
| **Latest version** | 6.0.2 (released August 2024) |
| **Python support** | Python >= 3.8 |
| **License** | MIT |
| **Repository** | https://github.com/yaml/pyyaml |
| **Documentation** | https://pyyaml.org/wiki/PyYAMLDocumentation |

The import name is `yaml`.

---

## Installation

```bash
pip install pyyaml
```

If the C library LibYAML is available on your system, PyYAML will use `CLoader`/`CDumper` for much faster parsing/emitting. The pure Python fallback is always available.

```python
import yaml
```

---

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

---

## Gotchas and Common Mistakes

### 1. The "Norway Problem" -- YAML 1.1 Boolean Coercion

In YAML 1.1, bare words like `yes`, `no`, `on`, `off`, `true`, `false` (and country codes like `NO` for Norway) are interpreted as booleans:

```python
yaml.safe_load("country: NO")   # {'country': False}  -- NOT "NO"!
yaml.safe_load("answer: yes")   # {'answer': True}
yaml.safe_load("port: on")      # {'port': True}
```

**Fix:** Quote strings that could be misinterpreted:

```yaml
country: "NO"
answer: "yes"
```

### 2. `safe_load()` Returns `None` for Empty Documents

```python
yaml.safe_load("")       # None (not an empty dict)
yaml.safe_load("---")    # None
yaml.safe_load("---\n")  # None
```

Always check for `None` when loading potentially empty files:

```python
data = yaml.safe_load(content) or {}
```

### 3. Integers with Leading Zeros Are Octal (YAML 1.1)

```python
yaml.safe_load("value: 0755")  # {'value': 493}  -- octal!
yaml.safe_load('value: "0755"')  # {'value': '0755'}  -- string
```

### 4. `safe_dump()` Sorts Keys by Default

```python
yaml.safe_dump({"b": 2, "a": 1})  # "a: 1\nb: 2\n" -- sorted!
yaml.safe_dump({"b": 2, "a": 1}, sort_keys=False)  # "b: 2\na: 1\n"
```

### 5. Multiline Strings

YAML has multiple multiline string styles. PyYAML auto-selects based on content:

```python
# Literal block (preserves newlines)
yaml.safe_load("""
text: |
  line one
  line two
""")
# {'text': 'line one\nline two\n'}

# Folded block (joins lines)
yaml.safe_load("""
text: >
  line one
  line two
""")
# {'text': 'line one line two\n'}
```

### 6. `safe_dump()` Cannot Serialize Custom Objects

```python
class Foo:
    pass

yaml.safe_dump(Foo())  # RepresenterError!
```

You must either add a custom representer or convert to basic types first.

### 7. Anchors and Aliases Can Cause Memory Issues

YAML supports anchors (`&anchor`) and aliases (`*anchor`) for referencing. Deeply nested or exponentially expanding alias structures (the "billion laughs" attack) can consume excessive memory:

```yaml
# Potentially dangerous with untrusted input
a: &a ["lol"]
b: &b [*a, *a]
c: &c [*b, *b]
```

`SafeLoader` does not prevent this. Consider input size limits for untrusted YAML.

---

## Complete Examples

### Example 1: Configuration File

```python
import yaml
from pathlib import Path

# Load config
config_path = Path("config.yaml")
with config_path.open() as f:
    config = yaml.safe_load(f) or {}

# Modify
config.setdefault("database", {})
config["database"]["host"] = "localhost"
config["database"]["port"] = 5432

# Save
with config_path.open("w") as f:
    yaml.safe_dump(config, f, sort_keys=False, default_flow_style=False)
```

### Example 2: Multi-Document Processing

```python
import yaml

yaml_stream = """
---
kind: Deployment
metadata:
  name: web
---
kind: Service
metadata:
  name: web-svc
---
kind: ConfigMap
metadata:
  name: web-config
"""

for doc in yaml.safe_load_all(yaml_stream):
    print(f"{doc['kind']}: {doc['metadata']['name']}")
# Deployment: web
# Service: web-svc
# ConfigMap: web-config
```

### Example 3: Using C-Accelerated Loader/Dumper

```python
import yaml

# Prefer C loader/dumper when available for performance
try:
    from yaml import CLoader as Loader, CDumper as Dumper
except ImportError:
    from yaml import SafeLoader as Loader, SafeDumper as Dumper

with open("large_file.yaml") as f:
    data = yaml.load(f, Loader=Loader)

output = yaml.dump(data, Dumper=Dumper)
```

### Example 4: Round-Trip Preserving Order

```python
import yaml

# Python 3.7+ dicts preserve insertion order
# Use sort_keys=False to maintain it through YAML

original = {"server": {"host": "0.0.0.0", "port": 8080}, "debug": True}
yaml_str = yaml.safe_dump(original, sort_keys=False)
restored = yaml.safe_load(yaml_str)

print(list(restored.keys()))  # ['server', 'debug'] -- order preserved
```

---

## See Also

- [PyYAML on GitHub](https://github.com/yaml/pyyaml)
- [PyYAML Documentation](https://pyyaml.org/wiki/PyYAMLDocumentation)
- [YAML 1.1 Specification](https://yaml.org/spec/1.1/)
- [YAML 1.2 Specification](https://yaml.org/spec/1.2/spec.html) -- note: PyYAML implements YAML 1.1
- [ruamel.yaml](https://yaml.readthedocs.io/) -- alternative that supports YAML 1.2 and round-trip preservation
