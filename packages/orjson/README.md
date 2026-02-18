# orjson

A **fast, correct JSON library** for Python, written in Rust.

`orjson` is a high-performance JSON serializer/deserializer that natively supports `datetime`, `date`, `time`, `UUID`, `dataclass`, `numpy` arrays, and more. It consistently benchmarks 2-10x faster than the standard library `json` module and other alternatives like `ujson`. A key distinction: `orjson.dumps()` returns `bytes`, not `str`.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `orjson` |
| **Latest version** | 3.10.15 (2025) |
| **Python support** | Python >= 3.8 |
| **License** | Apache 2.0 / MIT (dual) |
| **Repository** | https://github.com/ijl/orjson |
| **Documentation** | https://github.com/ijl/orjson#readme |

`orjson` is distributed as precompiled wheels for most platforms. No C compiler needed for installation.

---

## Installation

```bash
pip install orjson
```

```python
import orjson
```

---

## Core API Reference

### `orjson.dumps(obj, *, default=None, option=None)` -> `bytes`

Serialize a Python object to JSON. Returns **`bytes`**, not `str`.

```python
import orjson

data = {"name": "Alice", "scores": [95, 87, 92]}

# Returns bytes
result = orjson.dumps(data)
print(result)        # b'{"name":"Alice","scores":[95,87,92]}'
print(type(result))  # <class 'bytes'>

# If you need a str:
result_str = orjson.dumps(data).decode("utf-8")
```

### `orjson.loads(data)` -> `object`

Deserialize JSON from `bytes`, `bytearray`, `memoryview`, or `str`.

```python
import orjson

data = orjson.loads(b'{"name":"Alice","age":30}')
# {'name': 'Alice', 'age': 30}

# Also accepts str
data = orjson.loads('{"key": "value"}')
```

---

## Option Flags

Options are passed as a bitmask to `orjson.dumps()` via the `option` parameter. Combine multiple options with `|` (bitwise OR).

```python
orjson.dumps(data, option=orjson.OPT_INDENT_2 | orjson.OPT_SORT_KEYS)
```

### Available Options

| Option | Description |
|--------|-------------|
| `OPT_INDENT_2` | Pretty-print with 2-space indentation |
| `OPT_SORT_KEYS` | Sort dictionary keys alphabetically |
| `OPT_NON_STR_KEYS` | Allow non-string dict keys (int, float, bool, None, date, datetime, uuid, enum) |
| `OPT_NAIVE_UTC` | Serialize naive `datetime` objects as UTC (append `+00:00`) |
| `OPT_UTC_Z` | Use `Z` instead of `+00:00` for UTC timezone suffix |
| `OPT_OMIT_MICROSECONDS` | Omit microseconds from `datetime`/`time` serialization |
| `OPT_STRICT_INTEGER` | Raise `JSONEncodeError` for integers outside 53-bit range |
| `OPT_PASSTHROUGH_DATETIME` | Pass `datetime` objects to the `default` function instead of auto-serializing |
| `OPT_PASSTHROUGH_SUBCLASS` | Pass subclasses of serializable types to `default` instead of auto-serializing |
| `OPT_PASSTHROUGH_DATACLASS` | Pass `dataclass` instances to `default` instead of auto-serializing |
| `OPT_SERIALIZE_NUMPY` | Serialize `numpy` arrays and scalars |
| `OPT_SERIALIZE_UUID` | (Default behavior; included for completeness) |
| `OPT_APPEND_NEWLINE` | Append `\n` to the output |

### Examples with Options

```python
import orjson
from datetime import datetime, timezone

data = {"b": 2, "a": 1}

# Pretty-print with sorted keys
orjson.dumps(data, option=orjson.OPT_INDENT_2 | orjson.OPT_SORT_KEYS)
# b'{\n  "a": 1,\n  "b": 2\n}'

# Non-string keys
orjson.dumps({1: "one", 2: "two"}, option=orjson.OPT_NON_STR_KEYS)
# b'{"1":"one","2":"two"}'

# Naive datetime as UTC
dt = datetime(2025, 3, 15, 12, 0, 0)
orjson.dumps({"time": dt}, option=orjson.OPT_NAIVE_UTC)
# b'{"time":"2025-03-15T12:00:00+00:00"}'

# UTC with Z suffix
dt_utc = datetime(2025, 3, 15, 12, 0, 0, tzinfo=timezone.utc)
orjson.dumps({"time": dt_utc}, option=orjson.OPT_UTC_Z)
# b'{"time":"2025-03-15T12:00:00Z"}'

# Omit microseconds
dt_micro = datetime(2025, 3, 15, 12, 0, 0, 123456)
orjson.dumps({"time": dt_micro}, option=orjson.OPT_OMIT_MICROSECONDS)
# b'{"time":"2025-03-15T12:00:00"}'
```

---

## The `default` Function

For types that orjson does not natively support, provide a `default` callable. It receives the unsupported object and must return a serializable type.

```python
import orjson
from decimal import Decimal
from pathlib import Path

def default(obj):
    if isinstance(obj, Decimal):
        return float(obj)
    if isinstance(obj, Path):
        return str(obj)
    if isinstance(obj, set):
        return sorted(obj)
    raise TypeError(f"Object of type {type(obj)} is not JSON serializable")

data = {
    "price": Decimal("19.99"),
    "path": Path("/usr/local/bin"),
    "tags": {"python", "rust", "json"},
}

result = orjson.dumps(data, default=default)
# b'{"price":19.99,"path":"/usr/local/bin","tags":["json","python","rust"]}'
```

---

## Natively Supported Types

orjson has built-in fast-path serialization for these types (no `default` function needed):

| Type | JSON Output | Notes |
|------|-------------|-------|
| `str` | `"string"` | |
| `int` | `123` | 64-bit range by default |
| `float` | `1.23` | |
| `bool` | `true` / `false` | |
| `None` | `null` | |
| `list`, `tuple` | `[...]` | |
| `dict` | `{...}` | String keys by default; use `OPT_NON_STR_KEYS` for others |
| `datetime.datetime` | `"2025-03-15T12:00:00+00:00"` | ISO 8601 format |
| `datetime.date` | `"2025-03-15"` | |
| `datetime.time` | `"12:00:00"` | |
| `uuid.UUID` | `"550e8400-..."` | Canonical lowercase form |
| `dataclasses.dataclass` | `{...}` | Serialized as dict of fields |
| `enum.Enum` | value | Serializes the `.value` |
| `numpy.ndarray` | `[...]` | Requires `OPT_SERIALIZE_NUMPY` |
| `numpy` scalars | number | Requires `OPT_SERIALIZE_NUMPY` |

### Dataclass Example

```python
import orjson
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID

@dataclass
class User:
    id: UUID
    name: str
    created: datetime
    active: bool

user = User(
    id=UUID("12345678-1234-5678-1234-567812345678"),
    name="Alice",
    created=datetime(2025, 1, 15, 9, 30),
    active=True,
)

print(orjson.dumps(user, option=orjson.OPT_INDENT_2).decode())
# {
#   "id": "12345678-1234-5678-1234-567812345678",
#   "name": "Alice",
#   "created": "2025-01-15T09:30:00",
#   "active": true
# }
```

### Numpy Example

```python
import orjson
import numpy as np

arr = np.array([[1, 2, 3], [4, 5, 6]], dtype=np.int32)
result = orjson.dumps(arr, option=orjson.OPT_SERIALIZE_NUMPY)
# b'[[1,2,3],[4,5,6]]'

# Numpy scalars
orjson.dumps({"value": np.float64(3.14)}, option=orjson.OPT_SERIALIZE_NUMPY)
# b'{"value":3.14}'
```

---

## Performance Comparison

orjson is significantly faster than alternatives. Typical benchmarks (serialization of a complex nested structure):

| Library | Serialize | Deserialize |
|---------|-----------|-------------|
| `orjson` | ~1x (baseline) | ~1x (baseline) |
| `json` (stdlib) | ~5-10x slower | ~3-5x slower |
| `ujson` | ~2-4x slower | ~1.5-2x slower |
| `simplejson` | ~6-12x slower | ~3-6x slower |

Exact numbers vary by data shape, but orjson consistently leads in both serialization and deserialization.

---

## Gotchas and Common Mistakes

### 1. `dumps()` Returns `bytes`, Not `str`

This is the most common surprise for users migrating from `json.dumps()`:

```python
import orjson, json

json.dumps({"a": 1})    # '{"a": 1}'     -- str
orjson.dumps({"a": 1})  # b'{"a":1}'     -- bytes

# Convert if needed:
orjson.dumps({"a": 1}).decode("utf-8")  # str
```

Most web frameworks (Flask, FastAPI, Django) accept `bytes` for HTTP responses, so this is often a non-issue.

### 2. No `ensure_ascii` Option

orjson always outputs UTF-8. Non-ASCII characters are preserved, not escaped:

```python
orjson.dumps({"city": "Tokyo"})  # includes raw UTF-8 bytes for non-ASCII
```

### 3. Non-String Dict Keys Require `OPT_NON_STR_KEYS`

```python
# Raises TypeError by default
orjson.dumps({1: "one"})

# Works with option flag
orjson.dumps({1: "one"}, option=orjson.OPT_NON_STR_KEYS)
# b'{"1":"one"}'
```

### 4. No `cls` Parameter (Use `default` Instead)

Unlike `json.dumps(cls=MyEncoder)`, orjson uses a `default` function:

```python
# json style (not supported by orjson):
# json.dumps(data, cls=MyEncoder)

# orjson style:
orjson.dumps(data, default=my_default_func)
```

### 5. `OPT_PASSTHROUGH_DATETIME` for Custom Date Formatting

By default, orjson serializes `datetime` to ISO 8601. To use a custom format, pass it through to `default`:

```python
import orjson
from datetime import datetime

def default(obj):
    if isinstance(obj, datetime):
        return obj.strftime("%Y/%m/%d %H:%M")
    raise TypeError

orjson.dumps(
    {"time": datetime(2025, 3, 15, 12, 30)},
    default=default,
    option=orjson.OPT_PASSTHROUGH_DATETIME,
)
# b'{"time":"2025/03/15 12:30"}'
```

### 6. Integer Precision

By default, orjson serializes integers up to 64 bits. JavaScript can only safely handle 53-bit integers. Use `OPT_STRICT_INTEGER` to raise an error for values outside the safe range:

```python
orjson.dumps({"big": 2**53 + 1}, option=orjson.OPT_STRICT_INTEGER)
# Raises orjson.JSONEncodeError
```

---

## Complete Examples

### Example 1: FastAPI Response

```python
from fastapi import FastAPI
from fastapi.responses import Response
import orjson
from dataclasses import dataclass
from datetime import datetime

app = FastAPI()

@dataclass
class Item:
    name: str
    price: float
    updated: datetime

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    item = Item(name="Widget", price=9.99, updated=datetime.now())
    return Response(
        content=orjson.dumps(item),
        media_type="application/json",
    )
```

### Example 2: Drop-In Replacement for json

```python
import orjson

# Read JSON file
with open("data.json", "rb") as f:
    data = orjson.loads(f.read())

# Modify data
data["processed"] = True

# Write JSON file
with open("output.json", "wb") as f:
    f.write(orjson.dumps(data, option=orjson.OPT_INDENT_2))
```

### Example 3: Handling Mixed Types

```python
import orjson
from datetime import datetime, date
from uuid import uuid4
from decimal import Decimal
from enum import Enum
from dataclasses import dataclass

class Status(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"

@dataclass
class Record:
    id: str
    status: Status
    created: datetime

def default(obj):
    if isinstance(obj, Decimal):
        return str(obj)  # preserve precision as string
    raise TypeError

record = Record(
    id=str(uuid4()),
    status=Status.ACTIVE,
    created=datetime(2025, 3, 15),
)

payload = {
    "record": record,
    "amount": Decimal("199.95"),
    "date": date(2025, 3, 15),
}

output = orjson.dumps(payload, default=default, option=orjson.OPT_INDENT_2)
print(output.decode())
```

---

## See Also

- [orjson on GitHub](https://github.com/ijl/orjson)
- [orjson on PyPI](https://pypi.org/project/orjson/)
- [Benchmark Comparisons](https://github.com/ijl/orjson#performance) -- detailed benchmarks vs json, ujson, rapidjson
