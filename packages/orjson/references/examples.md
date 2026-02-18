# orjson — Examples & Gotchas

> Part of the orjson skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [dumps() Returns bytes, Not str](#1-dumps-returns-bytes-not-str)
  - [No ensure_ascii Option](#2-no-ensure_ascii-option)
  - [Non-String Dict Keys Require OPT_NON_STR_KEYS](#3-non-string-dict-keys-require-opt_non_str_keys)
  - [No cls Parameter](#4-no-cls-parameter-use-default-instead)
  - [OPT_PASSTHROUGH_DATETIME for Custom Date Formatting](#5-opt_passthrough_datetime-for-custom-date-formatting)
  - [Integer Precision](#6-integer-precision)
- [Complete Examples](#complete-examples)
  - [FastAPI Response](#example-1-fastapi-response)
  - [Drop-In Replacement for json](#example-2-drop-in-replacement-for-json)
  - [Handling Mixed Types](#example-3-handling-mixed-types)
- [See Also](#see-also)

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
