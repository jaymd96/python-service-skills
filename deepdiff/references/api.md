# DeepDiff — API Reference

> Part of the deepdiff skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API](#core-api)
  - [DeepDiff(t1, t2, ...)](#deepdifft1-t2-)
  - [DeepSearch(obj, item)](#deepsearchobj-item)
  - [DeepHash(obj)](#deephashobj)
  - [Delta(diff)](#deltadiff)
  - [extract(obj, path)](#extractobj-path)
- [Key Parameters](#key-parameters)
  - [ignore_order](#ignore_order)
  - [ignore_type_in_groups](#ignore_type_in_groups)
  - [exclude_paths](#exclude_paths)
  - [exclude_regex_paths](#exclude_regex_paths)
  - [exclude_types](#exclude_types)
  - [significant_digits](#significant_digits)
  - [ignore_string_case](#ignore_string_case)
  - [max_passes](#max_passes)
  - [max_diffs](#max_diffs)
  - [verbose_level](#verbose_level)
  - [view](#view)
  - [custom_operators](#custom_operators)
  - [ignore_nan_inequality](#ignore_nan_inequality)
  - [truncate_datetime](#truncate_datetime)
  - [ignore_private_variables](#ignore_private_variables)
  - [group_by](#group_by)
- [Tree View](#tree-view)
- [Serialization](#serialization)
- [Quick Reference](#quick-reference)

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
