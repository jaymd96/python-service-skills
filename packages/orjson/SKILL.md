---
name: orjson
description: Fast JSON serialization in Rust with native datetime/numpy/dataclass support. Use when needing high-performance JSON encoding/decoding, serializing datetime/UUID/dataclass/numpy natively, or replacing stdlib json. Triggers on orjson, fast JSON, JSON serialization, JSON performance, datetime JSON.
---

# orjson — Fast JSON (v3.10.15)

## Quick Start

```bash
pip install orjson
```

```python
import orjson

data = {"name": "Alice", "scores": [95, 87]}
encoded = orjson.dumps(data)        # returns bytes, not str
decoded = orjson.loads(encoded)     # accepts bytes or str
```

## Key Patterns

### dumps returns bytes (not str)
```python
result = orjson.dumps({"key": "value"})  # b'{"key":"value"}'
result_str = result.decode("utf-8")       # if you need str
```

### Option flags (combine with |)
```python
orjson.dumps(data, option=orjson.OPT_INDENT_2 | orjson.OPT_SORT_KEYS)
```

### Native type support (no custom encoder needed)
```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class User:
    name: str
    created: datetime

orjson.dumps(User("Alice", datetime.now()))
# dataclass, datetime, UUID, enum — all automatic
```

## References

- **[api.md](references/api.md)** — Full dumps/loads API, all option flags, default function, natively supported types, dataclass/numpy examples, performance benchmarks
- **[examples.md](references/examples.md)** — Common gotchas (bytes not str, non-string keys, no cls parameter, integer precision), FastAPI integration, drop-in json replacement, mixed type handling
