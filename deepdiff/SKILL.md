---
name: deepdiff
description: Deep comparison and diffing of Python objects. Use when comparing nested data structures, finding differences between objects, applying patches/deltas, or testing structural equality. Triggers on deep comparison, object diff, deepdiff, structural diff, data reconciliation, Delta patch.
---

# deepdiff — Object Diffing (v8.4.2)

## Quick Start

```bash
pip install deepdiff
```

```python
from deepdiff import DeepDiff

old = {"name": "Alice", "scores": [90, 85]}
new = {"name": "Alice", "scores": [90, 95], "role": "admin"}

diff = DeepDiff(old, new)
# {'values_changed': {"root['scores'][1]": {'new_value': 95, 'old_value': 85}},
#  'dictionary_item_added': ["root['role']"]}
```

## Key Patterns

### Common parameters
```python
DeepDiff(old, new,
    ignore_order=True,              # treat lists as sets
    exclude_paths=["root['id']"],   # skip specific paths
    significant_digits=2,           # float comparison precision
    ignore_string_case=True,        # case-insensitive
    verbose_level=2,                # include old/new values
)
```

### Delta (apply diffs as patches)
```python
from deepdiff import Delta

diff = DeepDiff(old, new)
delta = Delta(diff)
patched = old + delta  # apply diff to old → new
```

## References

- **[api.md](references/api.md)** — Full API for DeepDiff, DeepSearch, DeepHash, Delta, extract, all parameters, tree view, and serialization
- **[examples.md](references/examples.md)** — Gotchas, performance tips, and complete code examples
