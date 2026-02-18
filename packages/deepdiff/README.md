# DeepDiff

> Deep comparison, search, and hashing of Python objects -- perfect for reconciliation, testing, and patching.

---

## Overview

**DeepDiff** is a Python library for recursively comparing, searching, and hashing any Python object. It goes far beyond simple equality checks: it walks nested structures of dicts, lists, sets, tuples, custom objects, NumPy arrays, and more, producing a detailed, categorized report of every difference found. It is widely used in test suites, data reconciliation pipelines, configuration drift detection, and snapshot testing.

**Key Characteristics:**

- **Latest Stable Version:** 8.4.2 (released 2025; check [PyPI](https://pypi.org/project/deepdiff/) for the most current release)
- **Python:** 3.8+
- **License:** MIT
- **Dependencies:** `ordered-set` (required); `clevercsv`, `pyyaml` (optional, for serialization)
- **Repository:** [github.com/seperman/deepdiff](https://github.com/seperman/deepdiff)
- **Documentation:** [zepworks.com/deepdiff/current](https://zepworks.com/deepdiff/current/)

**Core Modules:**

| Module | Purpose |
|--------|---------|
| `DeepDiff` | Recursively compare two objects and produce a categorized diff |
| `DeepSearch` | Search within a nested object for a specific item |
| `DeepHash` | Generate a deterministic hash for any arbitrary Python object |
| `Delta` | Apply a diff as a patch to transform one object into another |
| `extract` | Extract a value from a nested object using a DeepDiff path string |

---

## Installation

```bash
# Standard install
pip install deepdiff

# With CLI support (adds command-line tools)
pip install "deepdiff[cli]"
```

Verify the installation:

```python
import deepdiff
print(deepdiff.__version__)
```

---

## Core API

### `DeepDiff(t1, t2, ...)`

Deep comparison of any two Python objects. `t1` is treated as the "old" or "reference" value and `t2` as the "new" or "current" value.

```python
from deepdiff import DeepDiff

t1 = {"name": "Alice", "age": 30, "scores": [90, 85, 92]}
t2 = {"name": "Alice", "age": 31, "scores": [90, 85, 95]}

diff = DeepDiff(t1, t2)
print(diff)
# {'values_changed': {"root['age']": {'new_value': 31, 'old_value': 30},
#                      "root['scores'][2]": {'new_value': 95, 'old_value': 92}}}
```

#### Result Categories

DeepDiff classifies every detected difference into one of these categories:

| Category | Description |
|----------|-------------|
| `type_changes` | Type of a value changed (e.g., `int` to `str`) |
| `values_changed` | Value changed but type stayed the same |
| `dictionary_item_added` | Key present in `t2` but not in `t1` |
| `dictionary_item_removed` | Key present in `t1` but not in `t2` |
| `iterable_item_added` | Item added to a list/tuple in `t2` |
| `iterable_item_removed` | Item removed from a list/tuple in `t2` |
| `set_item_added` | Item added to a set in `t2` |
| `set_item_removed` | Item removed from a set in `t2` |
| `attribute_added` | Attribute present on `t2` object but not on `t1` |
| `attribute_removed` | Attribute present on `t1` object but not on `t2` |
| `repetition_change` | Count of a repeated item changed (when `ignore_order=True`) |

```python
from deepdiff import DeepDiff

# Type change
diff = DeepDiff({"val": 1}, {"val": "1"})
print(diff)
# {'type_changes': {"root['val']": {'old_type': <class 'int'>,
#                                     'new_type': <class 'str'>,
#                                     'old_value': 1,
#                                     'new_value': '1'}}}

# Dictionary item added/removed
diff = DeepDiff({"a": 1, "b": 2}, {"a": 1, "c": 3})
print(diff)
# {'dictionary_item_added': ["root['c']"],
#  'dictionary_item_removed': ["root['b']"]}

# Set changes
diff = DeepDiff({1, 2, 3}, {2, 3, 4})
print(diff)
# {'set_item_added': [4], 'set_item_removed': [1]}
```

The diff object behaves like a dictionary. You can test for specific categories:

```python
diff = DeepDiff(t1, t2)

if "values_changed" in diff:
    print("Values were modified")

if not diff:
    print("Objects are identical")
```

---

### `DeepSearch(obj, item)`

Search within any nested object for a value or key.

```python
from deepdiff import DeepSearch

data = {
    "users": [
        {"name": "Alice", "role": "admin"},
        {"name": "Bob", "role": "user"},
    ],
    "metadata": {"created_by": "Alice"},
}

# Search for a value
result = DeepSearch(data, "Alice")
print(result)
# {'matched_values': ["root['users'][0]['name']", "root['metadata']['created_by']"]}

# Search for a key
result = DeepSearch(data, "role", verbose_level=2)
print(result)
# {'matched_keys': ["root['users'][0]['role']", "root['users'][1]['role']"]}
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `obj` | *(required)* | The object to search within |
| `item` | *(required)* | The value or key to search for |
| `exclude_paths` | `None` | Paths to skip during search |
| `exclude_types` | `None` | Types to skip during search |
| `verbose_level` | `2` | Controls amount of detail in results |
| `use_regexp` | `False` | Treat `item` as a regular expression |

```python
from deepdiff import DeepSearch

data = {"error_code": 404, "error_msg": "not found", "info": "OK"}

# Regex search
result = DeepSearch(data, "error.*", use_regexp=True)
print(result)
# {'matched_keys': ["root['error_code']", "root['error_msg']"]}
```

---

### `DeepHash(obj)`

Produce a deterministic hash of any Python object, including nested and mutable structures. This is useful for caching, deduplication, and change detection.

```python
from deepdiff import DeepHash

obj = {"name": "Alice", "scores": [90, 85, 92]}

hash_result = DeepHash(obj)

# The hash of the top-level object
print(hash_result[obj])  # A hex string like 'a1b2c3d4...'

# You can also hash individual components
print(hash_result[obj["name"]])
```

**Key parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ignore_order` | `False` | Ignore order in iterables for hashing |
| `significant_digits` | `None` | Precision for float hashing |
| `exclude_paths` | `None` | Paths to exclude from the hash |
| `exclude_types` | `None` | Types to exclude from the hash |
| `ignore_type_in_groups` | `None` | Groups of types to treat as equivalent |
| `apply_hash` | `True` | Whether to apply a final hash function (set `False` to see the intermediate representation) |

```python
from deepdiff import DeepHash

# Ignoring order -- same hash regardless of list ordering
h1 = DeepHash([1, 2, 3], ignore_order=True)
h2 = DeepHash([3, 1, 2], ignore_order=True)
assert h1[[1, 2, 3]] == h2[[3, 1, 2]]

# Float precision
h1 = DeepHash({"val": 3.14159}, significant_digits=2)
h2 = DeepHash({"val": 3.14}, significant_digits=2)
# These will hash to the same value at 2 significant digits
```

---

### `Delta(diff)`

Convert a DeepDiff result into a patch that can be applied to transform `t1` into `t2`. This enables workflows such as storing diffs, transmitting patches, and applying them later.

```python
from deepdiff import DeepDiff, Delta

t1 = {"name": "Alice", "age": 30, "scores": [90, 85]}
t2 = {"name": "Alice", "age": 31, "scores": [90, 85, 95]}

diff = DeepDiff(t1, t2)
delta = Delta(diff)

# Apply the delta to t1 to get t2
result = t1 + delta
print(result)
# {'name': 'Alice', 'age': 31, 'scores': [90, 85, 95]}
assert result == t2
```

**Serialization round-trip:**

```python
from deepdiff import DeepDiff, Delta

t1 = {"x": 1, "y": [1, 2]}
t2 = {"x": 2, "y": [1, 2, 3]}

diff = DeepDiff(t1, t2)

# Serialize to dict (JSON-safe)
diff_dict = diff.to_dict()

# Reconstruct a Delta from the serialized diff
delta = Delta(diff_dict)
result = t1 + delta
assert result == t2

# Or serialize the delta itself
delta_dumps = delta.dumps()  # bytes
delta_restored = Delta(delta_dumps)
result2 = t1 + delta_restored
assert result2 == t2
```

**Delta with flat dicts:**

```python
from deepdiff import Delta

# You can also create a Delta from a "flat" diff dictionary
delta = Delta({
    "values_changed": {
        "root['age']": {"new_value": 31}
    }
})
result = {"age": 30, "name": "Alice"} + delta
# {'age': 31, 'name': 'Alice'}
```

---

### `extract(obj, path)`

Extract a value from a nested object using a DeepDiff-style path string.

```python
from deepdiff import extract

data = {
    "users": [
        {"name": "Alice", "address": {"city": "Portland"}},
        {"name": "Bob", "address": {"city": "Seattle"}},
    ]
}

value = extract(data, "root['users'][1]['address']['city']")
print(value)  # 'Seattle'

value = extract(data, "root['users'][0]['name']")
print(value)  # 'Alice'
```

This is particularly useful when processing the paths returned by `DeepDiff` or `DeepSearch` results.

---

## Key Parameters

### `ignore_order`

When `True`, the order of items in lists and tuples is ignored. Items are compared by value rather than position.

```python
from deepdiff import DeepDiff

t1 = [1, 2, 3, 4]
t2 = [4, 3, 2, 1]

# With order (default)
diff = DeepDiff(t1, t2)
print(diff)
# {'values_changed': {'root[0]': {'new_value': 4, 'old_value': 1},
#                      'root[3]': {'new_value': 1, 'old_value': 4}}}

# Ignoring order
diff = DeepDiff(t1, t2, ignore_order=True)
print(diff)
# {}
```

### `ignore_type_in_groups`

Treat certain types as equivalent during comparison. Useful for comparing across numeric types or between strings and byte strings.

```python
from deepdiff import DeepDiff

t1 = {"count": 10}       # int
t2 = {"count": 10.0}     # float

# By default this is a type change
diff = DeepDiff(t1, t2)
print(diff)
# {'type_changes': {"root['count']": {'old_type': <class 'int'>, ...}}}

# Treat int and float as equivalent
diff = DeepDiff(t1, t2, ignore_type_in_groups=[(int, float)])
print(diff)
# {}
```

### `exclude_paths`

Skip specific paths during comparison.

```python
from deepdiff import DeepDiff

t1 = {"id": 1, "name": "Alice", "updated_at": "2025-01-01"}
t2 = {"id": 1, "name": "Alice", "updated_at": "2025-06-15"}

diff = DeepDiff(t1, t2, exclude_paths=["root['updated_at']"])
print(diff)
# {}
```

You can also pass paths without the `root` prefix:

```python
diff = DeepDiff(t1, t2, exclude_paths={"updated_at"})
print(diff)
# {}
```

### `exclude_regex_paths`

Skip paths matching regular expressions.

```python
from deepdiff import DeepDiff

t1 = {"user_id": 1, "user_name": "Alice", "data": {"user_created": "2025-01-01"}}
t2 = {"user_id": 2, "user_name": "Bob", "data": {"user_created": "2025-06-15"}}

# Exclude all paths containing 'user_'
diff = DeepDiff(t1, t2, exclude_regex_paths=[r"root\[.*user_.*\]"])
print(diff)
# {}
```

### `exclude_types`

Skip all values of specified types.

```python
from deepdiff import DeepDiff
import datetime

t1 = {"name": "Alice", "timestamp": datetime.datetime(2025, 1, 1)}
t2 = {"name": "Alice", "timestamp": datetime.datetime(2025, 6, 15)}

diff = DeepDiff(t1, t2, exclude_types=[datetime.datetime])
print(diff)
# {}
```

### `significant_digits`

Control the precision of float comparisons. The number specifies how many significant digits to consider.

```python
from deepdiff import DeepDiff

t1 = {"price": 3.14159265}
t2 = {"price": 3.14159}

# Default -- these are different
diff = DeepDiff(t1, t2)
print(diff)  # values_changed

# With significant_digits=5
diff = DeepDiff(t1, t2, significant_digits=5)
print(diff)
# {}
```

### `ignore_string_case`

Perform case-insensitive string comparison.

```python
from deepdiff import DeepDiff

t1 = {"name": "Alice"}
t2 = {"name": "alice"}

diff = DeepDiff(t1, t2, ignore_string_case=True)
print(diff)
# {}
```

### `max_passes`

Controls the maximum number of passes for the algorithm that pairs items when `ignore_order=True`. Higher values yield more accurate results at the cost of performance.

```python
from deepdiff import DeepDiff

# Default max_passes is 10000000
# Lower it if you have very large collections and can accept approximate results
diff = DeepDiff(large_list_1, large_list_2, ignore_order=True, max_passes=1000)
```

### `max_diffs`

Limits the total number of diffs returned. Useful for large objects where you only need to know whether differences exist rather than enumerate them all.

```python
from deepdiff import DeepDiff

diff = DeepDiff(big_obj_1, big_obj_2, max_diffs=100)
```

### `verbose_level`

Controls how much detail is included in the result.

| Level | Behavior |
|-------|----------|
| `0` | Minimum output |
| `1` | Adds `old_value` and `new_value` to `values_changed` |
| `2` (default) | Also adds the values for `type_changes`, adds items to `iterable_item_added`/`removed` |

```python
from deepdiff import DeepDiff

t1 = {"a": 1, "b": [1, 2]}
t2 = {"a": 2, "b": [1, 2, 3]}

diff = DeepDiff(t1, t2, verbose_level=0)
print(diff)
# {'values_changed': {"root['a']": {}}, 'iterable_item_added': ["root['b'][2]"]}

diff = DeepDiff(t1, t2, verbose_level=2)
print(diff)
# {'values_changed': {"root['a']": {'new_value': 2, 'old_value': 1}},
#  'iterable_item_added': {"root['b'][2]": 3}}
```

### `view`

Choose between `'text'` (default) and `'tree'` output formats.

```python
from deepdiff import DeepDiff

t1 = {"a": 1, "b": 2}
t2 = {"a": 10, "b": 2}

# Text view (default) -- paths as strings
diff = DeepDiff(t1, t2, view='text')
print(diff)
# {'values_changed': {"root['a']": {'new_value': 10, 'old_value': 1}}}

# Tree view -- paths as DiffLevel objects
diff = DeepDiff(t1, t2, view='tree')
for item in diff['values_changed']:
    print(f"Path: {item.path()}")
    print(f"  t1: {item.t1}")
    print(f"  t2: {item.t2}")
# Path: root['a']
#   t1: 1
#   t2: 10
```

### `custom_operators`

Extend comparison logic with your own operators. Create a class that inherits from `BaseOperator` and override the `give_up_diffing` method.

```python
from deepdiff import DeepDiff
from deepdiff.operator import BaseOperator


class RoundingOperator(BaseOperator):
    """Compare floats after rounding to a given number of decimal places."""

    def __init__(self, decimals=2, **kwargs):
        self.decimals = decimals
        super().__init__(**kwargs)

    def give_up_diffing(self, level, diff_instance):
        # Return True if you handled the comparison (no diff to report)
        # Return False if you want DeepDiff to handle it normally
        if isinstance(level.t1, float) and isinstance(level.t2, float):
            if round(level.t1, self.decimals) == round(level.t2, self.decimals):
                return True
        return False


t1 = {"price": 19.991}
t2 = {"price": 19.994}

diff = DeepDiff(
    t1, t2,
    custom_operators=[RoundingOperator(decimals=2, types=[float])]
)
print(diff)
# {}  (19.99 == 19.99 after rounding)
```

### `ignore_nan_inequality`

By default `float('nan') != float('nan')` in Python. Set this to `True` to treat NaN values as equal.

```python
from deepdiff import DeepDiff

t1 = {"val": float("nan")}
t2 = {"val": float("nan")}

diff = DeepDiff(t1, t2)
# Reports values_changed because nan != nan

diff = DeepDiff(t1, t2, ignore_nan_inequality=True)
print(diff)
# {}
```

### `truncate_datetime`

Truncate datetime comparisons to a specified granularity.

```python
from deepdiff import DeepDiff
import datetime

t1 = datetime.datetime(2025, 6, 15, 10, 30, 45)
t2 = datetime.datetime(2025, 6, 15, 10, 30, 59)

diff = DeepDiff(t1, t2, truncate_datetime="minute")
print(diff)
# {}
```

### `ignore_private_variables`

When `True` (the default), attributes starting with `_` are ignored when comparing objects.

```python
from deepdiff import DeepDiff

class Item:
    def __init__(self, name, _internal_id):
        self.name = name
        self._internal_id = _internal_id

a = Item("widget", 1001)
b = Item("widget", 1002)

diff = DeepDiff(a, b, ignore_private_variables=True)
print(diff)
# {}

diff = DeepDiff(a, b, ignore_private_variables=False)
print(diff)
# {'values_changed': {"root._internal_id": {'new_value': 1002, 'old_value': 1001}}}
```

### `group_by`

Group items in a list by a key before comparison. Extremely useful for comparing lists of dictionaries where order does not matter but a logical key identifies each item.

```python
from deepdiff import DeepDiff

t1 = [
    {"id": 1, "name": "Alice", "score": 90},
    {"id": 2, "name": "Bob", "score": 80},
]
t2 = [
    {"id": 2, "name": "Bob", "score": 85},
    {"id": 1, "name": "Alice", "score": 90},
]

# Without group_by, order matters and you get false diffs
diff = DeepDiff(t1, t2)
print(diff)
# values_changed detected because items are at different indices

# With group_by, items are matched by their 'id'
diff = DeepDiff(t1, t2, group_by="id")
print(diff)
# {'values_changed': {"root[2]['score']": {'new_value': 85, 'old_value': 80}}}
```

---

## Tree View

The tree view (`view='tree'`) returns `DiffLevel` objects that let you navigate the diff result programmatically. Each `DiffLevel` stores both sides of the comparison and path information.

```python
from deepdiff import DeepDiff

t1 = {
    "users": [
        {"name": "Alice", "age": 30},
        {"name": "Bob", "age": 25},
    ]
}
t2 = {
    "users": [
        {"name": "Alice", "age": 31},
        {"name": "Bob", "age": 26},
    ]
}

diff = DeepDiff(t1, t2, view='tree')

for category, items in diff.items():
    print(f"\n--- {category} ---")
    for item in items:
        print(f"  Path:      {item.path()}")
        print(f"  Old value: {item.t1}")
        print(f"  New value: {item.t2}")
        # Navigate up the tree
        print(f"  Parent t1: {item.up.t1}")
        print(f"  Parent t2: {item.up.t2}")
```

**DiffLevel attributes:**

| Attribute | Description |
|-----------|-------------|
| `t1` | The value from the first object at this level |
| `t2` | The value from the second object at this level |
| `path()` | The path string for the text view format |
| `path(output_format='list')` | The path as a list of keys/indices |
| `up` | The parent `DiffLevel` (one level up in the tree) |
| `down` | The child `DiffLevel` (one level down) |
| `report_type` | The category of the change (e.g., `'values_changed'`) |

```python
from deepdiff import DeepDiff

t1 = {"a": {"b": {"c": 1}}}
t2 = {"a": {"b": {"c": 2}}}

diff = DeepDiff(t1, t2, view='tree')

for item in diff['values_changed']:
    # Get the path as a list
    path_list = item.path(output_format='list')
    print(path_list)  # ['a', 'b', 'c']

    # Traverse the tree upward
    current = item
    while current is not None:
        print(f"  Level t1={current.t1}, t2={current.t2}")
        current = current.up
```

---

## Serialization

### Converting Diffs to JSON

```python
import json
from deepdiff import DeepDiff

t1 = {"name": "Alice", "scores": [90, 85]}
t2 = {"name": "Bob", "scores": [90, 85, 95]}

diff = DeepDiff(t1, t2)

# Method 1: to_dict() -- returns a JSON-serializable dictionary
diff_dict = diff.to_dict()
json_str = json.dumps(diff_dict, indent=2)
print(json_str)

# Method 2: to_json() -- directly returns a JSON string
json_str = diff.to_json()
print(json_str)
```

### Reconstructing Diffs from JSON

```python
import json
from deepdiff import DeepDiff

json_str = '{"values_changed": {"root[\'age\']": {"new_value": 31, "old_value": 30}}}'
diff_dict = json.loads(json_str)

# You can pass the dict to Delta to apply patches
from deepdiff import Delta

delta = Delta(diff_dict)
original = {"age": 30, "name": "Alice"}
patched = original + delta
print(patched)
# {'age': 31, 'name': 'Alice'}
```

### Delta Serialization

Delta objects can be serialized to bytes for storage or transmission:

```python
from deepdiff import DeepDiff, Delta

t1 = {"x": 1, "y": [1, 2]}
t2 = {"x": 2, "y": [1, 2, 3]}

# Create a delta
diff = DeepDiff(t1, t2)
delta = Delta(diff)

# Serialize to bytes
serialized = delta.dumps()

# Store, transmit, etc.

# Deserialize and apply
delta2 = Delta(serialized)
result = t1 + delta2
assert result == t2
```

### Delta to/from flat dictionaries

```python
from deepdiff import DeepDiff, Delta

t1 = {"a": 1, "b": {"c": 2}}
t2 = {"a": 1, "b": {"c": 3, "d": 4}}

diff = DeepDiff(t1, t2)

# Get the flat dict representation
flat = diff.to_dict()
print(flat)
# {'values_changed': {"root['b']['c']": {'new_value': 3, 'old_value': 2}},
#  'dictionary_item_added': {"root['b']['d']": 4}}

# Apply later
delta = Delta(flat)
assert (t1 + delta) == t2
```

---

## Gotchas and Common Mistakes

### 1. Performance with `ignore_order=True` on Large Collections

When `ignore_order=True`, DeepDiff must match each item in one iterable with items in the other, which can be O(n^2) in the worst case. For large lists (thousands of items), this can be very slow.

**Mitigations:**

```python
from deepdiff import DeepDiff

# Limit the effort
diff = DeepDiff(
    large_list_1, large_list_2,
    ignore_order=True,
    max_passes=1000,       # Limit matching passes
    max_diffs=100,          # Stop after finding 100 diffs
)

# Use group_by for lists of dicts with a unique key instead of ignore_order
diff = DeepDiff(
    list_of_dicts_1, list_of_dicts_2,
    group_by="id"           # Much faster than ignore_order for keyed records
)

# Convert to sets before comparing if items are hashable
diff = DeepDiff(set(list1), set(list2))
```

### 2. NumPy Array Handling

DeepDiff has built-in support for NumPy arrays, but there are subtleties:

```python
import numpy as np
from deepdiff import DeepDiff

a1 = np.array([1.0, 2.0, 3.0])
a2 = np.array([1.0, 2.0001, 3.0])

# Without significant_digits, tiny differences are reported
diff = DeepDiff(a1, a2)
print(diff)  # Reports values_changed at index 1

# Use significant_digits or math_epsilon for tolerance
diff = DeepDiff(a1, a2, significant_digits=3)
print(diff)  # {}

# math_epsilon for absolute tolerance
diff = DeepDiff(a1, a2, math_epsilon=0.001)
print(diff)  # {}
```

**Warning:** Using `ignore_order=True` with large NumPy arrays is extremely slow. Convert to sets or use NumPy's own comparison tools for unordered comparisons.

### 3. Custom `__eq__` Interaction

If your objects define `__eq__`, DeepDiff will still recurse into their attributes by default rather than relying solely on `__eq__`. This is by design -- DeepDiff wants to tell you *what* differs, not just *whether* things differ.

```python
from deepdiff import DeepDiff

class Money:
    def __init__(self, amount, currency):
        self.amount = amount
        self.currency = currency

    def __eq__(self, other):
        # Only compare amounts in the same currency
        return self.currency == other.currency and self.amount == other.amount

a = Money(100, "USD")
b = Money(100, "USD")
b._cache = "some_cached_value"  # internal attribute

diff = DeepDiff(a, b)
# DeepDiff will report _cache as attribute_added, even though a == b

# If you want to rely on __eq__ for certain types:
diff = DeepDiff(a, b, exclude_obj_callback=lambda obj, path: isinstance(obj, Money))
```

### 4. Path Format Differences Between Text and Tree View

Paths in text view are strings like `"root['a'][0]['b']"`. In tree view, they are `DiffLevel` objects with a `.path()` method.

```python
from deepdiff import DeepDiff

t1 = {"a": [{"b": 1}]}
t2 = {"a": [{"b": 2}]}

# Text view: paths are dict keys (strings)
diff_text = DeepDiff(t1, t2, view='text')
for path, detail in diff_text['values_changed'].items():
    print(type(path))   # <class 'str'>
    print(path)          # root['a'][0]['b']

# Tree view: values are DiffLevel objects
diff_tree = DeepDiff(t1, t2, view='tree')
for item in diff_tree['values_changed']:
    print(type(item))    # <class 'deepdiff.helper.DiffLevel'>
    print(item.path())   # root['a'][0]['b']
```

**Common mistake:** Trying to iterate over tree view results as a dictionary. In tree view, the values are sets of `DiffLevel` objects, not dictionaries.

### 5. Delta Limitations

- **Delta cannot handle all diff types equally.** Complex structural changes (such as simultaneously reordering and modifying list items) may not round-trip perfectly.
- **Custom objects:** Applying a delta to objects with custom `__init__` or `__setattr__` may fail or produce unexpected results.
- **Type changes:** When a value changes type (e.g., from a list to a dict), the delta replaces the entire value rather than merging.
- **Delta requires the base object:** You cannot apply a delta to an unrelated object and expect correct results; the delta encodes paths specific to the original structure.

```python
from deepdiff import DeepDiff, Delta

t1 = {"data": [1, 2, 3]}
t2 = {"data": {"a": 1}}  # Structural type change: list -> dict

diff = DeepDiff(t1, t2)
delta = Delta(diff)
result = t1 + delta
# result == {'data': {'a': 1}} -- the entire value is replaced
```

### 6. Mutability of Diff Results

The diff object is mutable. If you modify it, the changes stick. If you need the original, copy it or serialize it first.

```python
from deepdiff import DeepDiff
import copy

diff = DeepDiff({"a": 1}, {"a": 2})
diff_copy = copy.deepcopy(diff)
```

### 7. `ignore_order` with Nested Structures

When using `ignore_order=True` with deeply nested structures containing dicts inside lists, DeepDiff uses a heuristic to match items. Occasionally, it may pair items unexpectedly.

```python
from deepdiff import DeepDiff

t1 = [{"id": 1, "val": "a"}, {"id": 2, "val": "b"}]
t2 = [{"id": 2, "val": "b"}, {"id": 1, "val": "x"}]

# ignore_order tries to match items by similarity
diff = DeepDiff(t1, t2, ignore_order=True)
# Correctly identifies that id=1 had its val changed from 'a' to 'x'

# For predictable matching, prefer group_by:
diff = DeepDiff(t1, t2, group_by="id")
```

---

## Complete Code Examples

### Example 1: Comparing Nested Dicts

```python
from deepdiff import DeepDiff

config_v1 = {
    "database": {
        "host": "localhost",
        "port": 5432,
        "credentials": {"user": "admin", "password": "secret"},
    },
    "cache": {"ttl": 300, "backend": "redis"},
    "features": ["auth", "logging"],
}

config_v2 = {
    "database": {
        "host": "db.prod.internal",
        "port": 5432,
        "credentials": {"user": "admin", "password": "new_secret"},
        "ssl": True,
    },
    "cache": {"ttl": 600, "backend": "redis"},
    "features": ["auth", "logging", "metrics"],
}

diff = DeepDiff(config_v1, config_v2)

print("=== Configuration Drift Report ===\n")

if "values_changed" in diff:
    print("Modified values:")
    for path, change in diff["values_changed"].items():
        print(f"  {path}:")
        print(f"    was: {change['old_value']}")
        print(f"    now: {change['new_value']}")

if "dictionary_item_added" in diff:
    print("\nNew settings:")
    for path in diff["dictionary_item_added"]:
        print(f"  {path}")

if "iterable_item_added" in diff:
    print("\nNew list items:")
    for path, value in diff["iterable_item_added"].items():
        print(f"  {path}: {value}")
```

Output:
```
=== Configuration Drift Report ===

Modified values:
  root['database']['host']:
    was: localhost
    now: db.prod.internal
  root['database']['credentials']['password']:
    was: secret
    now: new_secret
  root['cache']['ttl']:
    was: 300
    now: 600

New settings:
  root['database']['ssl']

New list items:
  root['features'][2]: metrics
```

### Example 2: Comparing Objects with Attributes

```python
from deepdiff import DeepDiff


class Server:
    def __init__(self, hostname, ip, port, tags):
        self.hostname = hostname
        self.ip = ip
        self.port = port
        self.tags = tags

    def __repr__(self):
        return f"Server({self.hostname})"


server_v1 = Server("web-01", "10.0.1.5", 8080, {"env": "staging", "role": "web"})
server_v2 = Server("web-01", "10.0.1.10", 8443, {"env": "production", "role": "web"})

diff = DeepDiff(server_v1, server_v2)
print(diff)
# {'values_changed': {"root.ip": {'new_value': '10.0.1.10', 'old_value': '10.0.1.5'},
#                      "root.port": {'new_value': 8443, 'old_value': 8080},
#                      "root.tags['env']": {'new_value': 'production', 'old_value': 'staging'}}}

# Only compare specific attributes
diff = DeepDiff(
    server_v1, server_v2,
    exclude_paths=["root.ip", "root.port"]
)
print(diff)
# Only shows the tags['env'] change
```

### Example 3: Ignoring Order in Lists

```python
from deepdiff import DeepDiff

api_response_v1 = {
    "status": "ok",
    "results": [
        {"id": 1, "name": "Widget A", "price": 9.99},
        {"id": 2, "name": "Widget B", "price": 14.99},
        {"id": 3, "name": "Widget C", "price": 4.99},
    ],
}

api_response_v2 = {
    "status": "ok",
    "results": [
        {"id": 3, "name": "Widget C", "price": 4.99},
        {"id": 1, "name": "Widget A", "price": 10.99},
        {"id": 2, "name": "Widget B", "price": 14.99},
    ],
}

# With ignore_order, the reordering is ignored; only the price change is reported
diff = DeepDiff(api_response_v1, api_response_v2, ignore_order=True)
print(diff)
# {'values_changed': {"root['results'][0]['price']": {'new_value': 10.99, 'old_value': 9.99}}}

# Even better: use group_by for predictable matching
diff = DeepDiff(api_response_v1, api_response_v2, group_by="id",
                group_by_sort_key="id")
print(diff)
```

### Example 4: Using `exclude_paths` for Selective Comparison

```python
from deepdiff import DeepDiff
import datetime


def compare_records(record1, record2):
    """Compare two records, ignoring metadata fields."""
    diff = DeepDiff(
        record1,
        record2,
        exclude_paths=[
            "root['_id']",
            "root['created_at']",
            "root['updated_at']",
            "root['_version']",
        ],
        exclude_types=[datetime.datetime],
        ignore_string_case=True,
    )
    return diff


r1 = {
    "_id": "abc123",
    "_version": 1,
    "created_at": datetime.datetime(2025, 1, 1),
    "updated_at": datetime.datetime(2025, 1, 1),
    "name": "Alice Smith",
    "email": "alice@example.com",
    "role": "Admin",
}

r2 = {
    "_id": "def456",
    "_version": 3,
    "created_at": datetime.datetime(2025, 6, 15),
    "updated_at": datetime.datetime(2025, 6, 15),
    "name": "Alice Smith",
    "email": "ALICE@EXAMPLE.COM",
    "role": "admin",
}

diff = compare_records(r1, r2)
print(diff)
# {} -- all meaningful fields are equivalent after exclusions and case folding
```

### Example 5: Applying Delta Patches

```python
from deepdiff import DeepDiff, Delta
import json


def create_migration(old_config, new_config):
    """Create a serializable migration patch between two configurations."""
    diff = DeepDiff(old_config, new_config)
    delta = Delta(diff)
    return delta.dumps()  # Serialized bytes


def apply_migration(config, migration_bytes):
    """Apply a migration patch to a configuration."""
    delta = Delta(migration_bytes)
    return config + delta


# Create the initial config
config_v1 = {
    "app_name": "MyApp",
    "debug": True,
    "database": {"host": "localhost", "port": 5432},
    "allowed_origins": ["http://localhost:3000"],
}

# The target config
config_v2 = {
    "app_name": "MyApp",
    "debug": False,
    "database": {"host": "db.prod.internal", "port": 5432, "ssl": True},
    "allowed_origins": ["https://myapp.com", "https://api.myapp.com"],
}

# Create and store the migration
migration = create_migration(config_v1, config_v2)
print(f"Migration size: {len(migration)} bytes")

# Later, apply the migration
result = apply_migration(config_v1, migration)
assert result == config_v2
print("Migration applied successfully!")
print(json.dumps(result, indent=2))
```

### Example 6: Custom Operators for Domain-Specific Comparison

```python
from deepdiff import DeepDiff
from deepdiff.operator import BaseOperator
import math


class CoordinateProximityOperator(BaseOperator):
    """Consider two (lat, lon) tuples equal if within a given distance in km."""

    def __init__(self, max_distance_km=1.0, **kwargs):
        self.max_distance_km = max_distance_km
        super().__init__(**kwargs)

    def _haversine(self, lat1, lon1, lat2, lon2):
        R = 6371  # Earth radius in km
        dlat = math.radians(lat2 - lat1)
        dlon = math.radians(lon2 - lon1)
        a = (math.sin(dlat / 2) ** 2 +
             math.cos(math.radians(lat1)) *
             math.cos(math.radians(lat2)) *
             math.sin(dlon / 2) ** 2)
        return R * 2 * math.asin(math.sqrt(a))

    def give_up_diffing(self, level, diff_instance):
        if isinstance(level.t1, tuple) and isinstance(level.t2, tuple):
            if len(level.t1) == 2 and len(level.t2) == 2:
                dist = self._haversine(
                    level.t1[0], level.t1[1],
                    level.t2[0], level.t2[1]
                )
                if dist <= self.max_distance_km:
                    return True  # Close enough, no diff
        return False  # Let DeepDiff handle it


location_v1 = {
    "name": "Office",
    "coords": (45.5231, -122.6765),  # Portland, OR
}

location_v2 = {
    "name": "Office",
    "coords": (45.5235, -122.6770),  # Very nearby
}

diff = DeepDiff(
    location_v1,
    location_v2,
    custom_operators=[CoordinateProximityOperator(max_distance_km=1.0, types=[tuple])]
)
print(diff)
# {} -- the coordinates are within 1km of each other


class VersionCompatibilityOperator(BaseOperator):
    """Consider two version strings equal if they share the same major.minor."""

    def give_up_diffing(self, level, diff_instance):
        if isinstance(level.t1, str) and isinstance(level.t2, str):
            try:
                major1, minor1, *_ = level.t1.split(".")
                major2, minor2, *_ = level.t2.split(".")
                if major1 == major2 and minor1 == minor2:
                    return True
            except ValueError:
                pass
        return False


deps_v1 = {"requests": "2.31.0", "flask": "3.0.1"}
deps_v2 = {"requests": "2.31.2", "flask": "3.1.0"}

diff = DeepDiff(
    deps_v1, deps_v2,
    custom_operators=[VersionCompatibilityOperator(types=[str])]
)
print(diff)
# Only reports flask change (3.0 -> 3.1), not requests (2.31.0 -> 2.31.2)
```

### Example 7: Integration with Testing and Assertions

```python
import unittest
from deepdiff import DeepDiff


class TestAPIResponse(unittest.TestCase):
    """Use DeepDiff for detailed test assertions."""

    def assertDeepEqual(self, expected, actual, **kwargs):
        """Custom assertion that provides detailed diff on failure."""
        diff = DeepDiff(expected, actual, **kwargs)
        if diff:
            self.fail(
                f"Objects are not deeply equal.\n"
                f"Diff:\n{diff.to_json(indent=2)}"
            )

    def test_user_endpoint(self):
        expected = {
            "id": 1,
            "name": "Alice",
            "email": "alice@example.com",
            "roles": ["admin", "user"],
        }

        # Simulated API response
        actual = {
            "id": 1,
            "name": "Alice",
            "email": "alice@example.com",
            "roles": ["user", "admin"],  # Different order
        }

        # This would fail without ignore_order
        self.assertDeepEqual(expected, actual, ignore_order=True)

    def test_partial_comparison(self):
        """Compare only the fields we care about."""
        expected_subset = {"status": "active", "plan": "premium"}

        actual = {
            "id": 42,
            "status": "active",
            "plan": "premium",
            "created_at": "2025-01-01T00:00:00Z",
            "internal_flags": {"migrated": True},
        }

        diff = DeepDiff(
            expected_subset,
            actual,
            exclude_paths=[
                "root['id']",
                "root['created_at']",
                "root['internal_flags']",
            ],
        )
        self.assertFalse(diff, f"Unexpected differences: {diff}")


# --- Pytest style ---

def test_snapshot_comparison():
    """Compare object against a known snapshot."""
    from deepdiff import DeepDiff

    snapshot = {
        "users": [
            {"name": "Alice", "score": 95.0},
            {"name": "Bob", "score": 87.5},
        ]
    }

    result = {
        "users": [
            {"name": "Alice", "score": 95.001},
            {"name": "Bob", "score": 87.499},
        ]
    }

    diff = DeepDiff(snapshot, result, significant_digits=1)
    assert not diff, f"Snapshot mismatch: {diff}"


def test_with_exclude_regex():
    """Ignore all timestamp-like fields using regex."""
    from deepdiff import DeepDiff

    obj1 = {
        "data": {"value": 42, "fetched_at": "2025-01-01"},
        "meta": {"processed_at": "2025-01-01", "version": 1},
    }
    obj2 = {
        "data": {"value": 42, "fetched_at": "2025-06-15"},
        "meta": {"processed_at": "2025-06-15", "version": 1},
    }

    diff = DeepDiff(obj1, obj2, exclude_regex_paths=[r".*_at"])
    assert not diff
```

---

## Quick Reference

### Constructor Signatures

```python
from deepdiff import DeepDiff, DeepSearch, DeepHash, Delta, extract

# DeepDiff -- compare two objects
diff = DeepDiff(
    t1, t2,
    ignore_order=False,
    report_repetition=False,
    significant_digits=None,
    number_format_notation="f",
    exclude_paths=None,
    exclude_regex_paths=None,
    exclude_types=None,
    ignore_type_in_groups=None,
    ignore_string_case=False,
    ignore_nan_inequality=False,
    ignore_private_variables=True,
    truncate_datetime=None,
    verbose_level=2,
    view='text',
    max_passes=10000000,
    max_diffs=None,
    custom_operators=None,
    group_by=None,
    math_epsilon=None,
    log_frequency_in_sec=0,
)

# DeepSearch -- search within an object
result = DeepSearch(
    obj, item,
    exclude_paths=None,
    exclude_regex_paths=None,
    exclude_types=None,
    verbose_level=2,
    use_regexp=False,
)

# DeepHash -- hash an object
hashes = DeepHash(
    obj,
    ignore_order=False,
    significant_digits=None,
    exclude_paths=None,
    exclude_types=None,
    ignore_type_in_groups=None,
    apply_hash=True,
)

# Delta -- create a patch from a diff
delta = Delta(
    diff,                    # DeepDiff result, dict, or serialized bytes
    raise_errors=False,      # raise on application errors
    bidirectional=False,     # allow applying delta in reverse
    force=False,             # force apply even if structure doesn't match
)

# extract -- get a value by path
value = extract(obj, path)   # path is a string like "root['a'][0]['b']"
```

### Diff Result Structure (Text View)

```python
{
    "type_changes": {
        "root['key']": {
            "old_type": "<class 'int'>",
            "new_type": "<class 'str'>",
            "old_value": 1,
            "new_value": "1"
        }
    },
    "values_changed": {
        "root['key']": {
            "new_value": 2,
            "old_value": 1
        }
    },
    "dictionary_item_added": ["root['new_key']"],
    "dictionary_item_removed": ["root['old_key']"],
    "iterable_item_added": {
        "root['list'][2]": "new_item"
    },
    "iterable_item_removed": {
        "root['list'][0]": "removed_item"
    },
    "set_item_added": ["root[4]"],
    "set_item_removed": ["root[1]"],
    "attribute_added": ["root.new_attr"],
    "attribute_removed": ["root.old_attr"],
    "repetition_change": {
        "root[0]": {
            "old_repeat": 2,
            "new_repeat": 3,
            "old_indexes": [0, 1],
            "new_indexes": [0, 1, 2],
            "value": "item"
        }
    }
}
```

---

## Further Resources

- **Official Documentation:** [zepworks.com/deepdiff/current](https://zepworks.com/deepdiff/current/)
- **PyPI:** [pypi.org/project/deepdiff](https://pypi.org/project/deepdiff/)
- **GitHub:** [github.com/seperman/deepdiff](https://github.com/seperman/deepdiff)
- **Changelog:** [github.com/seperman/deepdiff/blob/master/CHANGELOG.md](https://github.com/seperman/deepdiff/blob/master/CHANGELOG.md)
