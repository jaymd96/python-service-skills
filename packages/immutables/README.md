# immutables -- High-Performance Immutable Mapping for Python

## Overview

**immutables** is a Python library providing a high-performance immutable mapping type based on a Hash Array Mapped Trie (HAMT) data structure. It is developed by [MagicStack](https://github.com/MagicStack), the same team behind `uvloop` and `asyncpg`.

The core type, `immutables.Map`, is a persistent (in the functional programming sense) immutable dictionary that supports efficient structural sharing. When you "modify" a Map, you get back a new Map that shares most of its internal structure with the original, making copies extremely cheap in both time and memory.

**Key Characteristics:**

| Attribute | Detail |
|-----------|--------|
| **Version** | 0.21 (latest stable as of early 2026) |
| **Python** | Requires 3.8+ |
| **License** | Apache 2.0 |
| **Implementation** | C extension (with pure-Python fallback) |
| **Dependencies** | None |
| **Thread Safety** | Yes (immutable objects are inherently thread-safe) |
| **Used Internally By** | CPython `contextvars` module (PEP 567) |

**Why immutables?**

- Persistent data structure with O(log32 N) operations -- effectively constant time
- Structural sharing means creating "modified copies" is far cheaper than copying a `dict`
- C extension provides performance competitive with built-in `dict` for reads
- Battle-tested: CPython's own `contextvars.Context` uses the same HAMT implementation internally
- Enables functional programming patterns, undo/redo systems, and safe concurrent state

---

## Installation

```bash
pip install immutables
```

To verify the installation:

```python
import immutables
print(immutables.__version__)
```

The package ships with prebuilt wheels for most platforms. If a wheel is not available, a C compiler is required to build the extension. The library includes a pure-Python fallback that is used automatically if the C extension cannot be compiled.

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

## Gotchas and Common Mistakes

### 1. Not a Drop-in `dict` Replacement

`immutables.Map` deliberately does **not** implement `collections.abc.MutableMapping`. You cannot use `m[key] = value` or `del m[key]` on a Map. This is by design -- the immutable API forces you to acknowledge that operations return new objects.

```python
m = immutables.Map(a=1)

# WRONG -- this raises TypeError
# m["b"] = 2

# CORRECT -- use set() and capture the return value
m2 = m.set("b", 2)
```

A common mistake is forgetting to capture the return value:

```python
m = immutables.Map(a=1)

# BUG: the return value is discarded!
m.set("b", 2)
print("b" in m)  # False -- m is unchanged

# CORRECT
m = m.set("b", 2)
print("b" in m)  # True
```

### 2. `mutate()` / `finish()` Lifecycle

The `MapMutation` object becomes invalid after `finish()` is called. Using it afterward raises an error:

```python
m = immutables.Map()
mm = m.mutate()
mm["a"] = 1
result = mm.finish()

# WRONG -- mutation is already finished
# mm["b"] = 2  # Raises an error
```

Use the context manager form to make the lifecycle explicit:

```python
m = immutables.Map()
with m.mutate() as mm:
    mm["a"] = 1
    mm["b"] = 2
    result = mm.finish()
# mm is out of scope here -- no accidental reuse
```

### 3. No Built-in JSON Serialization

`immutables.Map` is not recognized by Python's `json` module:

```python
import json
import immutables

m = immutables.Map(name="Alice", age=30)

# WRONG -- raises TypeError: Object of type Map is not JSON serializable
# json.dumps(m)

# CORRECT -- convert to dict first
json.dumps(dict(m.items()))

# Or use a custom encoder
class MapEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, immutables.Map):
            return dict(obj.items())
        return super().default(obj)

json.dumps(m, cls=MapEncoder)
```

### 4. Iteration Order is Not Insertion Order

Unlike `dict` (which preserves insertion order since Python 3.7), `immutables.Map` iteration order is determined by key hashes and the HAMT structure. Do not rely on any particular order.

```python
m = immutables.Map(z=1, a=2, m=3)
# Iteration order may NOT be z, a, m
print(list(m.keys()))  # Could be any permutation
```

### 5. When a Plain `dataclass` or `NamedTuple` is Better

If you have a fixed, known set of fields (e.g., a configuration struct), a frozen `dataclass` or `NamedTuple` is simpler, faster, and more type-safe:

```python
from dataclasses import dataclass

# If your "map" always has these specific fields, prefer this:
@dataclass(frozen=True)
class Config:
    host: str
    port: int
    debug: bool = False

# Over this:
# config = immutables.Map(host="localhost", port=8080, debug=False)
```

Use `immutables.Map` when you need a **dynamic** set of keys, when keys are not known at development time, or when you need structural sharing across versions.

### 6. Nested Mutability

Immutability is shallow. If you store mutable objects as values, those objects can still be modified:

```python
m = immutables.Map(data=[1, 2, 3])
m["data"].append(4)  # This mutates the list inside the "immutable" map!
print(m["data"])  # [1, 2, 3, 4] -- the map didn't protect the list

# For deep immutability, store only immutable values (tuples, frozensets, etc.)
m = immutables.Map(data=(1, 2, 3))
```

---

## Complete Code Examples

### Example 1: Basic Map Operations

```python
import immutables

# --- Creating maps ---
empty = immutables.Map()
from_kwargs = immutables.Map(host="localhost", port=5432, dbname="mydb")
from_dict = immutables.Map({"x": 10, "y": 20})
from_pairs = immutables.Map([("r", 255), ("g", 128), ("b", 0)])

# --- Reading values ---
print(from_kwargs["host"])           # "localhost"
print(from_kwargs.get("timeout"))    # None
print(from_kwargs.get("timeout", 30))  # 30

# --- Setting values (returns new map) ---
config = immutables.Map(debug=False, log_level="INFO")
config_v2 = config.set("debug", True)
config_v3 = config_v2.set("log_level", "DEBUG")

print(config["debug"])      # False (original unchanged)
print(config_v3["debug"])   # True

# --- Deleting keys ---
config_v4 = config_v3.delete("debug")
print("debug" in config_v4)  # False
print("debug" in config_v3)  # True (previous version unchanged)

# --- Iteration ---
for key, value in config_v3.items():
    print(f"  {key}: {value}")

# --- Equality ---
a = immutables.Map(x=1, y=2)
b = immutables.Map(y=2, x=1)
print(a == b)  # True -- same content

# --- Using as dict key (hashable) ---
cache = {}
cache[a] = "result for this configuration"
print(cache[b])  # "result for this configuration" -- a == b, same hash
```

### Example 2: Batch Mutations with `mutate()`

```python
import immutables

# Start with some initial data
initial = immutables.Map(version=1, author="system")

# Batch-add many entries efficiently using mutate()
with initial.mutate() as mm:
    mm["title"] = "Introduction to HAMT"
    mm["tags"] = ("python", "data-structures", "immutable")
    mm["version"] = 2  # overwrite existing key
    mm["word_count"] = 1500
    mm["published"] = True
    article = mm.finish()

print(f"Article version: {article['version']}")  # 2
print(f"Initial version: {initial['version']}")  # 1 (unchanged)

# Batch-delete with mutate()
with article.mutate() as mm:
    del mm["word_count"]
    del mm["published"]
    draft = mm.finish()

print("published" in article)  # True (original unchanged)
print("published" in draft)    # False

# Conditional mutations
with article.mutate() as mm:
    if mm.get("published"):
        mm["status"] = "live"
    else:
        mm["status"] = "draft"
    result = mm.finish()

print(result["status"])  # "live"
```

### Example 3: Building Up a Map Incrementally

```python
import immutables

def parse_headers(raw_headers: list[tuple[str, str]]) -> immutables.Map:
    """Parse a list of HTTP headers into an immutable Map.

    Uses mutate() for efficient batch construction.
    """
    m = immutables.Map()
    with m.mutate() as mm:
        for name, value in raw_headers:
            key = name.lower()
            existing = mm.get(key)
            if existing is not None:
                # HTTP allows multiple values for the same header
                mm[key] = existing + ", " + value
            else:
                mm[key] = value
        return mm.finish()

headers = parse_headers([
    ("Content-Type", "application/json"),
    ("Accept", "text/html"),
    ("Accept", "application/json"),
    ("Authorization", "Bearer token123"),
    ("X-Request-ID", "abc-def-ghi"),
])

print(headers["content-type"])  # "application/json"
print(headers["accept"])        # "text/html, application/json"
print(len(headers))             # 4


def merge_maps(*maps: immutables.Map) -> immutables.Map:
    """Merge multiple immutable Maps, with later maps taking precedence."""
    if not maps:
        return immutables.Map()

    result = maps[0]
    with result.mutate() as mm:
        for m in maps[1:]:
            for key, value in m.items():
                mm[key] = value
        return mm.finish()

defaults = immutables.Map(timeout=30, retries=3, debug=False)
overrides = immutables.Map(timeout=60, debug=True)
env_config = immutables.Map(api_key="secret", region="us-east-1")

final = merge_maps(defaults, overrides, env_config)
print(dict(final.items()))
# {'timeout': 60, 'retries': 3, 'debug': True, 'api_key': 'secret', 'region': 'us-east-1'}
```

### Example 4: State Management with Structural Sharing

```python
import immutables
from dataclasses import dataclass, field
from typing import Any


@dataclass
class StateManager:
    """A simple state manager demonstrating structural sharing.

    Each state transition creates a new immutable Map version.
    Old versions are retained for history/debugging, and structural
    sharing ensures this is memory-efficient.
    """
    _state: immutables.Map = field(default_factory=immutables.Map)
    _history: list[immutables.Map] = field(default_factory=list)
    _listeners: list = field(default_factory=list)

    @property
    def state(self) -> immutables.Map:
        return self._state

    def dispatch(self, action: str, payload: dict[str, Any]) -> immutables.Map:
        """Apply an action to the state, returning the new state."""
        old_state = self._state

        if action == "SET":
            with old_state.mutate() as mm:
                for k, v in payload.items():
                    mm[k] = v
                self._state = mm.finish()

        elif action == "DELETE":
            with old_state.mutate() as mm:
                for key in payload.get("keys", []):
                    if key in mm:
                        del mm[key]
                self._state = mm.finish()

        elif action == "RESET":
            self._state = immutables.Map()

        else:
            raise ValueError(f"Unknown action: {action}")

        self._history.append(old_state)
        self._notify_listeners(old_state, self._state)
        return self._state

    def get_history(self) -> list[immutables.Map]:
        """Return all previous states (cheap -- they share structure)."""
        return list(self._history)

    def rollback(self, steps: int = 1) -> immutables.Map:
        """Roll back to a previous state."""
        for _ in range(min(steps, len(self._history))):
            if self._history:
                self._state = self._history.pop()
        return self._state

    def subscribe(self, listener):
        self._listeners.append(listener)

    def _notify_listeners(self, old_state, new_state):
        for listener in self._listeners:
            listener(old_state, new_state)


# Usage
store = StateManager()

# Subscribe to state changes
def on_change(old, new):
    print(f"State changed: {len(list(old.items()))} -> {len(list(new.items()))} entries")

store.subscribe(on_change)

# Dispatch actions
store.dispatch("SET", {"user": "Alice", "role": "admin", "theme": "dark"})
store.dispatch("SET", {"theme": "light", "language": "en"})
store.dispatch("DELETE", {"keys": ["language"]})

print(f"\nCurrent state:")
for k, v in store.state.items():
    print(f"  {k}: {v}")

print(f"\nHistory has {len(store.get_history())} snapshots")

# Rollback
store.rollback(2)
print(f"\nAfter rollback:")
for k, v in store.state.items():
    print(f"  {k}: {v}")

# All historical states are intact and memory-efficient
# thanks to structural sharing in the HAMT
```

---

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

---

## Quick Reference

```python
import immutables

# Create
m = immutables.Map()                        # empty
m = immutables.Map(a=1, b=2)                # from kwargs
m = immutables.Map({"a": 1, "b": 2})        # from dict
m = immutables.Map([("a", 1), ("b", 2)])     # from pairs

# Read
v = m["key"]                                 # get (KeyError if missing)
v = m.get("key", default)                    # get with default

# "Modify" (returns new Map)
m2 = m.set("key", "value")                  # set/update a key
m2 = m.delete("key")                         # remove a key

# Batch modify
with m.mutate() as mm:
    mm["a"] = 1
    mm["b"] = 2
    del mm["old_key"]
    m2 = mm.finish()

# Inspect
len(m)                                       # number of entries
"key" in m                                   # containment check
list(m.keys())                               # all keys
list(m.values())                             # all values
list(m.items())                              # all (key, value) pairs

# Compare
m1 == m2                                     # equality
hash(m)                                      # hashable (if values are)

# Convert
dict(m.items())                              # to dict
```

---

## Further Reading

- **GitHub Repository:** https://github.com/MagicStack/immutables
- **PyPI Page:** https://pypi.org/project/immutables/
- **PEP 567 -- Context Variables:** https://peps.python.org/pep-0567/
- **HAMT Wikipedia:** https://en.wikipedia.org/wiki/Hash_array_mapped_trie
- **Phil Bagwell's original HAMT paper:** "Ideal Hash Trees" (2001)
- **Yury Selivanov's PyCon talk:** "Immutables and contextvars" -- explains the design decisions behind the library
