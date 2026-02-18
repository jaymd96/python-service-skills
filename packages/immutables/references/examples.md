# immutables — Examples & Gotchas

> Part of the immutables skill. See [SKILL.md](../SKILL.md) for overview.

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Not a Drop-in dict Replacement](#1-not-a-drop-in-dict-replacement)
  - [2. mutate() / finish() Lifecycle](#2-mutate--finish-lifecycle)
  - [3. No Built-in JSON Serialization](#3-no-built-in-json-serialization)
  - [4. Iteration Order is Not Insertion Order](#4-iteration-order-is-not-insertion-order)
  - [5. When a Plain dataclass or NamedTuple is Better](#5-when-a-plain-dataclass-or-namedtuple-is-better)
  - [6. Nested Mutability](#6-nested-mutability)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic Map Operations](#example-1-basic-map-operations)
  - [Example 2: Batch Mutations with mutate()](#example-2-batch-mutations-with-mutate)
  - [Example 3: Building Up a Map Incrementally](#example-3-building-up-a-map-incrementally)
  - [Example 4: State Management with Structural Sharing](#example-4-state-management-with-structural-sharing)
- [Quick Reference](#quick-reference)
- [Further Reading](#further-reading)

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

## Further Reading

- **GitHub Repository:** https://github.com/MagicStack/immutables
- **PyPI Page:** https://pypi.org/project/immutables/
- **PEP 567 -- Context Variables:** https://peps.python.org/pep-0567/
- **HAMT Wikipedia:** https://en.wikipedia.org/wiki/Hash_array_mapped_trie
- **Phil Bagwell's original HAMT paper:** "Ideal Hash Trees" (2001)
- **Yury Selivanov's PyCon talk:** "Immutables and contextvars" -- explains the design decisions behind the library
