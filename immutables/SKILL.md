---
name: immutables
description: High-performance immutable mapping (HAMT) for Python. Use when needing persistent data structures with structural sharing, immutable maps for state management, or efficient copy-on-write semantics. Triggers on immutable map, HAMT, persistent data structure, immutables, structural sharing.
---

# immutables — Immutable Maps (v0.21)

## Quick Start

```bash
pip install immutables
```

```python
from immutables import Map

m = Map()
m2 = m.set("key", "value")  # returns NEW map
m3 = m2.set("b", 2)
m3["key"]  # "value"
len(m3)    # 2
m is not m2  # True — original unchanged
```

## Key Patterns

### Batch mutations (efficient bulk updates)
```python
with m.mutate() as mm:
    mm["a"] = 1
    mm["b"] = 2
    mm["c"] = 3
    del mm["old_key"]
    new_map = mm.finish()
```

### Structural sharing (undo/redo)
```python
history = [Map()]
current = history[-1]
current = current.set("x", 1)
history.append(current)
# Undo: history.pop(); current = history[-1]
# Memory efficient — shared internal nodes
```

## References

- **[api.md](references/api.md)** — Core API (Map, set/delete/mutate), MapMutation, HAMT internals, performance characteristics, use cases
- **[examples.md](references/examples.md)** — Complete examples, gotchas, quick reference, further reading
