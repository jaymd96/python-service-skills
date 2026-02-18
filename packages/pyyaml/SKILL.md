---
name: pyyaml
description: YAML parser and emitter for Python. Use when reading/writing YAML configuration files, serializing Python objects to YAML, or processing YAML documents. Triggers on yaml, pyyaml, YAML parsing, YAML loading, YAML dumping, configuration files, yaml.safe_load.
---

# PyYAML — YAML Parser (v6.0.2)

## Quick Start

```bash
pip install pyyaml
```

```python
import yaml

data = yaml.safe_load("""
name: Alice
languages:
  - Python
  - Rust
""")
# {'name': 'Alice', 'languages': ['Python', 'Rust']}
```

## Key Patterns

### Loading (ALWAYS use safe_load)
```python
data = yaml.safe_load(open("config.yaml"))
docs = list(yaml.safe_load_all(open("multi.yaml")))  # multiple documents
```

### Dumping
```python
yaml.safe_dump(data, sort_keys=False, default_flow_style=False)

# To file
with open("out.yaml", "w") as f:
    yaml.safe_dump(data, f)
```

## References

- **[api.md](references/api.md)** — Security warnings, loader/dumper classes, safe_load/safe_dump parameters, custom representers and constructors
- **[examples.md](references/examples.md)** — Common gotchas (Norway problem, empty docs, octal ints), config file patterns, multi-document processing, C-accelerated usage
