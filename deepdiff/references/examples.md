# DeepDiff — Examples & Gotchas

> Part of the deepdiff skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Performance with ignore_order=True on Large Collections](#1-performance-with-ignore_ordertrue-on-large-collections)
  - [2. NumPy Array Handling](#2-numpy-array-handling)
  - [3. Custom __eq__ Interaction](#3-custom-__eq__-interaction)
  - [4. Path Format Differences Between Text and Tree View](#4-path-format-differences-between-text-and-tree-view)
  - [5. Delta Limitations](#5-delta-limitations)
  - [6. Mutability of Diff Results](#6-mutability-of-diff-results)
  - [7. ignore_order with Nested Structures](#7-ignore_order-with-nested-structures)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Comparing Nested Dicts](#example-1-comparing-nested-dicts)
  - [Example 2: Comparing Objects with Attributes](#example-2-comparing-objects-with-attributes)
  - [Example 3: Ignoring Order in Lists](#example-3-ignoring-order-in-lists)
  - [Example 4: Using exclude_paths for Selective Comparison](#example-4-using-exclude_paths-for-selective-comparison)
  - [Example 5: Applying Delta Patches](#example-5-applying-delta-patches)
  - [Example 6: Custom Operators for Domain-Specific Comparison](#example-6-custom-operators-for-domain-specific-comparison)
  - [Example 7: Integration with Testing and Assertions](#example-7-integration-with-testing-and-assertions)
- [Further Resources](#further-resources)

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

## Further Resources

- **Official Documentation:** [zepworks.com/deepdiff/current](https://zepworks.com/deepdiff/current/)
- **PyPI:** [pypi.org/project/deepdiff](https://pypi.org/project/deepdiff/)
- **GitHub:** [github.com/seperman/deepdiff](https://github.com/seperman/deepdiff)
- **Changelog:** [github.com/seperman/deepdiff/blob/master/CHANGELOG.md](https://github.com/seperman/deepdiff/blob/master/CHANGELOG.md)
