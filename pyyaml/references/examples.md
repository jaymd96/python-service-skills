# PyYAML — Examples & Gotchas

> Part of the pyyaml skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [The "Norway Problem"](#1-the-norway-problem----yaml-11-boolean-coercion)
  - [safe_load() Returns None for Empty Documents](#2-safe_load-returns-none-for-empty-documents)
  - [Integers with Leading Zeros Are Octal](#3-integers-with-leading-zeros-are-octal-yaml-11)
  - [safe_dump() Sorts Keys by Default](#4-safe_dump-sorts-keys-by-default)
  - [Multiline Strings](#5-multiline-strings)
  - [safe_dump() Cannot Serialize Custom Objects](#6-safe_dump-cannot-serialize-custom-objects)
  - [Anchors and Aliases Can Cause Memory Issues](#7-anchors-and-aliases-can-cause-memory-issues)
- [Complete Examples](#complete-examples)
  - [Configuration File](#example-1-configuration-file)
  - [Multi-Document Processing](#example-2-multi-document-processing)
  - [Using C-Accelerated Loader/Dumper](#example-3-using-c-accelerated-loaderdumper)
  - [Round-Trip Preserving Order](#example-4-round-trip-preserving-order)
- [See Also](#see-also)

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
