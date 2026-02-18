# immutables — API Reference

> Part of the immutables skill. See [SKILL.md](../SKILL.md) for overview.

- [Core API](#core-api)
  - [Creating a Map](#creating-a-map)
  - [map.set(key, value) -- Setting Values](#mapsetkey-value----setting-values)
  - [map.delete(key) -- Removing Keys](#mapdeletekey----removing-keys)
  - [Accessing Values](#accessing-values)
  - [map.mutate() -- Batch Mutations with MapMutation](#mapmutate----batch-mutations-with-mapmutation)
  - [MapMutation API](#mapmutation-api)
  - [Iteration, Length, and Containment](#iteration-length-and-containment)
  - [Equality and Hashing](#equality-and-hashing)
  - [map.update() -- Merging Maps](#mapupdate----merging-maps)
- [HAMT Internals](#hamt-internals)
- [Performance Characteristics](#performance-characteristics)
- [Use Cases](#use-cases)
- [Type Hints](#type-hints)

## Core API

### Creating a Map

```python
import immutables

# Create an empty Map
m = immutables.Map()

# Create a Map from keyword arguments
m = immutables.Map(name="Alice", age=30, role="engineer")

# Create a Map from an iterable of key-value pairs
m = immutables.Map([("x", 1), ("y", 2), ("z", 3)])

# Create from another mapping (dict, etc.)
m = immutables.Map({"host": "localhost", "port": 8080})
```

`Map()` accepts the same constructor arguments as `dict()`: keyword arguments, an iterable of `(key, value)` pairs, or another mapping object.

### `map.set(key, value)` -- Setting Values

```python
m1 = immutables.Map()

# set() returns a NEW Map -- the original is unchanged
m2 = m1.set("name", "Alice")
m3 = m2.set("age", 30)

print(m1)  # Map({})
print(m2)  # Map({'name': 'Alice'})
print(m3)  # Map({'name': 'Alice', 'age': 30})
```

**Important:** `set()` does not mutate the original Map. It returns a brand-new Map that shares internal structure with the original. This is the fundamental property of a persistent data structure.

To update an existing key, call `set()` with the same key:

```python
m4 = m3.set("age", 31)
print(m3["age"])  # 30 -- original unchanged
print(m4["age"])  # 31 -- new map has updated value
```

### `map.delete(key)` -- Removing Keys

```python
m = immutables.Map(x=1, y=2, z=3)

m2 = m.delete("y")
print(m2)  # Map({'x': 1, 'z': 3})
print(m)   # Map({'x': 1, 'y': 2, 'z': 3}) -- unchanged
```

Raises `KeyError` if the key does not exist:

```python
try:
    m.delete("nonexistent")
except KeyError as e:
    print(f"Key not found: {e}")
```

### Accessing Values

```python
m = immutables.Map(name="Alice", age=30)

# Bracket notation -- raises KeyError if missing
name = m["name"]  # "Alice"

# .get() with optional default -- returns None if missing (like dict)
age = m.get("age")         # 30
missing = m.get("email")   # None
missing = m.get("email", "N/A")  # "N/A"
```

### `map.mutate()` -- Batch Mutations with MapMutation

For building up or transforming a Map with many changes, calling `set()` repeatedly creates an intermediate Map on every call. The `mutate()` context manager provides a **transient mutable builder** (`MapMutation`) that applies many changes efficiently and then produces a single new immutable Map.

```python
m = immutables.Map()

with m.mutate() as mm:
    mm["name"] = "Alice"
    mm["age"] = 30
    mm["role"] = "engineer"
    mm["department"] = "platform"
    new_map = mm.finish()

print(new_map)
# Map({'name': 'Alice', 'age': 30, 'role': 'engineer', 'department': 'platform'})
print(m)
# Map({}) -- original is still empty
```

**Key points about MapMutation:**

- `mm[key] = value` sets a key (like `dict`)
- `del mm[key]` deletes a key (raises `KeyError` if missing)
- `mm.set(key, value)` is also available as an alternative to bracket assignment
- `mm.get(key, default)` and `key in mm` work as expected
- `mm.finish()` returns the final immutable `Map` and invalidates the mutation object
- After calling `finish()`, the `MapMutation` can no longer be used

#### Using mutate() Without a Context Manager

You can also use `mutate()` without the `with` statement, but you **must** call `finish()` to get the result:

```python
m = immutables.Map(x=1)
mm = m.mutate()
mm["y"] = 2
mm["z"] = 3
new_map = mm.finish()
```

The context manager form is preferred because it makes the lifecycle clear and avoids accidentally using the mutation object after `finish()`.

### MapMutation API

The `MapMutation` object returned by `map.mutate()` supports:

| Operation | Syntax | Description |
|-----------|--------|-------------|
| Set | `mm[key] = value` or `mm.set(key, value)` | Set a key-value pair |
| Delete | `del mm[key]` | Remove a key (raises `KeyError` if missing) |
| Get | `mm[key]` or `mm.get(key, default)` | Retrieve a value |
| Contains | `key in mm` | Check if a key exists |
| Length | `len(mm)` | Number of key-value pairs |
| Finish | `mm.finish()` | Return final immutable Map |

### Iteration, Length, and Containment

`immutables.Map` supports the standard mapping iteration protocols:

```python
m = immutables.Map(a=1, b=2, c=3)

# Length
print(len(m))  # 3

# Containment
print("a" in m)      # True
print("z" in m)      # False

# Iterate over keys (default)
for key in m:
    print(key)

# Iterate over keys explicitly
for key in m.keys():
    print(key)

# Iterate over values
for value in m.values():
    print(value)

# Iterate over key-value pairs
for key, value in m.items():
    print(f"{key} = {value}")
```

**Note:** Iteration order is **not** guaranteed to be insertion order (unlike `dict` in Python 3.7+). The order is determined by the internal HAMT structure and hash values of the keys.

### Equality and Hashing

```python
m1 = immutables.Map(a=1, b=2)
m2 = immutables.Map(b=2, a=1)

# Maps with the same content are equal
print(m1 == m2)  # True

# Maps can be used as dictionary keys or in sets (they are hashable)
s = {m1, m2}
print(len(s))  # 1 -- same content, same hash
```

Maps are hashable only if all their keys and values are hashable. If a Map contains unhashable values, attempting to hash it will raise a `TypeError`.

### `map.update()` -- Merging Maps

```python
m1 = immutables.Map(a=1, b=2)

# Update with keyword arguments
m2 = m1.update(b=20, c=30)
print(m2)  # Map({'a': 1, 'b': 20, 'c': 30})

# Update with a dict or iterable of pairs
m3 = m1.update({"c": 3, "d": 4})
print(m3)  # Map({'a': 1, 'b': 2, 'c': 3, 'd': 4})
```

Like `set()`, `update()` returns a new Map. The original is unchanged.

## HAMT Internals

### What is a Hash Array Mapped Trie?

A **Hash Array Mapped Trie (HAMT)** is a tree-based data structure that uses the hash of each key to determine the key's position in the tree. The hash is consumed in small chunks (typically 5 bits at a time, yielding a branching factor of 32) as you descend the trie.

```
Root (dispatch on bits 0-4 of hash)
├── Node 0
│   ├── Node 0-0 (bits 5-9)
│   │   └── Leaf: ("name", "Alice")
│   └── Node 0-7
│       └── Leaf: ("age", 30)
├── Node 5
│   └── Leaf: ("role", "engineer")
└── Node 19
    ├── Leaf: ("dept", "platform")
    └── Node 19-3 (collision resolution)
        ├── Leaf: ("team", "infra")
        └── Leaf: ("org", "eng")
```

### Why HAMT is Efficient for Immutable Maps

1. **Structural Sharing:** When you "modify" a HAMT (e.g., inserting a key), only the nodes along the path from the root to the affected leaf need to be copied. All other nodes are shared with the original tree. With a branching factor of 32, a tree holding millions of entries is only ~7 levels deep (log32(1,000,000) ~ 4). This means a `set()` operation copies at most ~7 nodes, regardless of the map size.

2. **Compact Representation:** HAMT nodes use bitmap-indexed arrays. Instead of allocating a full 32-element array at each node (which would be wasteful for sparse nodes), a 32-bit bitmap indicates which slots are occupied, and only occupied slots consume memory.

3. **Cache Friendliness:** The wide branching factor (32) keeps the tree shallow, reducing pointer chasing. Each node's children are stored in a contiguous array, improving CPU cache utilization.

4. **Hash Collision Handling:** When two keys have identical hashes, HAMT stores them in a special collision node that uses linear search -- but collisions are rare with good hash functions.

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `map[key]` (get) | O(log32 N) | Effectively O(1) for practical N |
| `map.set(key, value)` | O(log32 N) | Returns new Map; copies ~7 nodes max |
| `map.delete(key)` | O(log32 N) | Returns new Map |
| `len(map)` | O(1) | Size is tracked |
| `key in map` | O(log32 N) | Same as get |
| Iteration | O(N) | Full tree traversal |
| `map.mutate()` batch | O(K * log32 N) | K = number of mutations |

Because log32(N) grows extremely slowly, these operations are effectively constant time for all practical data sizes:

| N (entries) | log32(N) (max depth) |
|-------------|---------------------|
| 32 | 1 |
| 1,024 | 2 |
| 32,768 | 3 |
| 1,048,576 | 4 |
| 33,554,432 | 5 |
| 1,073,741,824 | 6 |

### Memory Efficiency Through Structural Sharing

When you create a modified copy of a `dict`, you must copy every key-value pair, consuming O(N) time and memory. With `immutables.Map`, a "copy" shares almost all of its structure with the original:

```python
import immutables

# This is a large map
big_map = immutables.Map()
with big_map.mutate() as mm:
    for i in range(100_000):
        mm[f"key_{i}"] = i
    big_map = mm.finish()

# Creating a "modified copy" is cheap -- only ~7 nodes are new
big_map_v2 = big_map.set("key_50000", "updated")

# Both maps exist simultaneously, sharing >99.99% of memory
```

### Comparison with Alternatives

| Feature | `immutables.Map` | `dict` copy | `types.MappingProxyType` | `frozenset` of tuples |
|---------|------------------|-------------|--------------------------|----------------------|
| Immutable | Yes | No (copy is mutable) | Read-only view (underlying dict can change) | Yes (but not a mapping) |
| Copy cost | O(log32 N) per change | O(N) full copy | O(1) wrapping, but no modification | O(N) to construct |
| Structural sharing | Yes | No | N/A (view, not copy) | No |
| Hashable | Yes (if values hashable) | No | No | Yes |
| Lookup speed | ~2-5x slower than dict | Same as dict | Same as dict | O(N) linear scan |
| Memory (per version) | Shared structure | Full copy each | Shared with source dict | Full copy |
| Thread safe | Yes (immutable) | No | Read-only (but source can change) | Yes (frozen) |
| C implementation | Yes | Yes (built-in) | Yes (built-in) | Yes (built-in) |

**When to use each:**

- **`immutables.Map`**: You need multiple versions of a mapping to coexist efficiently (undo/redo, functional state management, snapshots)
- **`dict.copy()`**: You need a one-off copy and do not care about structural sharing
- **`types.MappingProxyType`**: You want to expose a read-only view of a dict that you control internally
- **`frozendict` (third party)**: You need a hashable dict but do not need structural sharing or persistent modifications

## Use Cases

### 1. Undo/Redo Systems

Because each `set()` or `delete()` returns a new Map while the old one remains intact, building an undo/redo stack is trivial:

```python
import immutables

class UndoableState:
    def __init__(self):
        self._history: list[immutables.Map] = [immutables.Map()]
        self._position: int = 0

    @property
    def current(self) -> immutables.Map:
        return self._history[self._position]

    def update(self, key: str, value):
        """Apply an update, discarding any redo history."""
        new_state = self.current.set(key, value)
        # Discard everything after current position
        self._history = self._history[:self._position + 1]
        self._history.append(new_state)
        self._position += 1

    def undo(self):
        if self._position > 0:
            self._position -= 1

    def redo(self):
        if self._position < len(self._history) - 1:
            self._position += 1

# Usage
state = UndoableState()
state.update("name", "Alice")
state.update("age", 30)
state.update("age", 31)

print(state.current["age"])  # 31
state.undo()
print(state.current["age"])  # 30
state.undo()
print(state.current.get("age"))  # None -- only "name" exists
state.redo()
print(state.current["age"])  # 30
```

Each state in the history is a full, independent snapshot that shares structure with its neighbors. Storing 1,000 versions of a 10,000-element map costs far less memory than 1,000 full copies.

### 2. Persistent Data Structures for Functional Programming

```python
import immutables
from typing import TypeVar, Callable

T = TypeVar("T")

def transform_map(
    m: immutables.Map,
    key: str,
    fn: Callable[[T], T]
) -> immutables.Map:
    """Pure function: returns a new map with fn applied to key's value."""
    old_value = m.get(key)
    if old_value is not None:
        return m.set(key, fn(old_value))
    return m

# Functional pipeline -- each step produces a new version
config = immutables.Map(retries=3, timeout=30, debug=False)
config_v2 = transform_map(config, "retries", lambda r: r + 1)
config_v3 = transform_map(config_v2, "timeout", lambda t: t * 2)

# All three versions exist independently
print(config["retries"])     # 3
print(config_v2["retries"])  # 4
print(config_v3["timeout"])  # 60
```

### 3. Thread-Safe Shared State

Because `immutables.Map` is truly immutable, it can be safely shared across threads without locks:

```python
import immutables
import threading

# Shared state -- safe to read from any thread
shared_config: immutables.Map = immutables.Map(
    db_host="localhost",
    db_port=5432,
    pool_size=10
)

def worker(config: immutables.Map, worker_id: int):
    """Each worker reads the immutable config without any locking."""
    host = config["db_host"]
    port = config["db_port"]
    print(f"Worker {worker_id}: connecting to {host}:{port}")

threads = []
for i in range(10):
    t = threading.Thread(target=worker, args=(shared_config, i))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# To "update" the config, swap the reference atomically
# (Python's GIL makes simple reference assignment atomic)
shared_config = shared_config.set("pool_size", 20)
```

### 4. contextvars (Python's Context Uses Immutables Internally)

Python 3.7+ introduced `contextvars` (PEP 567) for managing context-local state in asyncio applications. Under the hood, `contextvars.Context` uses the exact same HAMT data structure from the `immutables` package. This is why `immutables` was created in the first place -- it was extracted from the CPython implementation of `contextvars`.

```python
import contextvars
import immutables

# contextvars uses HAMT internally for efficient copy-on-write
# When you copy a Context (e.g., in asyncio.Task), it uses structural
# sharing to avoid duplicating the entire context.

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id")

# Each asyncio task gets its own Context (backed by HAMT),
# efficiently sharing state with the parent context.
```

The `immutables` library gives you direct access to this same powerful data structure for your own application-level state management.

## Type Hints

`immutables.Map` supports generic type parameters for static type checking:

```python
import immutables

# Type-annotated maps
config: immutables.Map[str, int] = immutables.Map(retries=3, timeout=30)
headers: immutables.Map[str, str] = immutables.Map()

# Works with mypy, pyright, etc.
def get_setting(settings: immutables.Map[str, int], key: str) -> int:
    return settings.get(key, 0)
```
