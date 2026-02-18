---
name: toolz
description: Functional programming primitives for Python — pipe, curry, compose, and data transformations. Use when building data pipelines, composing functions, currying, or using FP patterns like groupby/valmap. Triggers on functional programming, pipe, curry, compose, toolz, FP utilities, data pipeline.
---

# toolz — FP Primitives (v1.0.0)

## Quick Start

```bash
pip install toolz
# pip install cytoolz  # C-accelerated, 2-5x faster
```

```python
from toolz import pipe, curry, groupby, valmap

result = pipe(
    data,
    lambda d: filter(is_valid, d),
    lambda d: map(transform, d),
    list,
)
```

## Key Patterns

### pipe (left-to-right composition)
```python
from toolz import pipe
result = pipe(raw_data, parse, validate, transform, serialize)
```

### curry (partial application)
```python
from toolz import curry

@curry
def add(x, y):
    return x + y

add5 = add(5)
add5(3)  # 8
```

### groupby + valmap (split-apply-combine)
```python
from toolz import groupby, valmap
groups = groupby(lambda u: u["role"], users)
counts = valmap(len, groups)
```

## References

- **[api.md](references/api.md)** — Full itertoolz/functoolz/dicttoolz API (60+ functions), key patterns, cytoolz, curried module
- **[examples.md](references/examples.md)** — Complete examples, gotchas, performance notes
