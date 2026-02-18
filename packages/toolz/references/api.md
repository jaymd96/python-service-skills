# toolz — API Reference

> Part of the toolz skill. See [SKILL.md](../SKILL.md) for overview.

- [Core API by Category](#core-api-by-category)
  - [Itertoolz](#itertoolz)
  - [Functoolz](#functoolz)
  - [Dicttoolz](#dicttoolz)
- [Key Patterns](#key-patterns)
  - [pipe() for Data Pipeline Composition](#pipe-for-data-pipeline-composition)
  - [curry() for Partial Application](#curry-for-partial-application)
  - [compose() vs compose_left() vs pipe()](#compose-vs-compose_left-vs-pipe)
  - [groupby() + valmap() for Data Transformation](#groupby--valmap-for-data-transformation)
  - [memoize() for Caching](#memoize-for-caching)
  - [excepts() for Functional Error Handling](#excepts-for-functional-error-handling)
- [cytoolz -- Cython-Accelerated Version](#cytoolz----cython-accelerated-version)
- [toolz.curried Module Reference](#toolzcurried-module-reference)

## Core API by Category

### Itertoolz

Functions for working with iterables. Most return **lazy iterators** (not lists).

| Function | Signature | Description |
|----------|-----------|-------------|
| `accumulate` | `accumulate(binop, seq, initial='__no__default__')` | Running reduction. Like `itertools.accumulate` but takes the function first. |
| `concat` | `concat(seqs)` | Concatenate zero or more iterables (lazy `itertools.chain.from_iterable`). |
| `concatv` | `concatv(*seqs)` | Variadic version of `concat`. `concatv([1], [2])` instead of `concat([[1], [2]])`. |
| `cons` | `cons(el, seq)` | Prepend element to an iterable. `cons(1, [2, 3])` -> `(1, 2, 3)`. |
| `count` | `count(seq)` | Count elements in an iterable (consumes it). |
| `drop` | `drop(n, seq)` | Drop the first `n` elements from a sequence. |
| `first` | `first(seq)` | Return the first element. |
| `frequencies` | `frequencies(seq)` | Count occurrences of each element. Returns a dict. |
| `get` | `get(ind, seq, default='__no__default__')` | Get element(s) by index. Supports multiple indices: `get([0, 2], 'abcde')` -> `('a', 'c')`. |
| `groupby` | `groupby(key, seq)` | Group elements by a key function. Returns `{key: [elements]}`. |
| `interleave` | `interleave(seqs)` | Interleave elements from multiple iterables. `interleave([[1,2],[3,4]])` -> `(1,3,2,4)`. |
| `interpose` | `interpose(el, seq)` | Insert element between each pair. `interpose('x', [1,2,3])` -> `(1,'x',2,'x',3)`. |
| `isdistinct` | `isdistinct(seq)` | Return True if all elements are unique. |
| `isiterable` | `isiterable(x)` | Return True if `x` is iterable (but not a string). |
| `iterate` | `iterate(func, x)` | Infinite sequence: `x, func(x), func(func(x)), ...` |
| `join` | `join(leftkey, leftseq, rightkey, rightseq, left_default='__no__default__', right_default='__no__default__')` | Join two sequences like a SQL join on key functions. |
| `last` | `last(seq)` | Return the last element. |
| `mapcat` | `mapcat(func, seqs)` | Map then concatenate. `mapcat(f, seqs)` == `concat(map(f, seqs))`. |
| `merge_sorted` | `merge_sorted(*seqs, key=None)` | Merge pre-sorted iterables into a single sorted iterable. |
| `nth` | `nth(n, seq)` | Return the nth element (0-indexed). |
| `partition` | `partition(n, seq, pad='__no__default__')` | Split into tuples of length `n`. Drops incomplete tail unless `pad` given. |
| `partition_all` | `partition_all(n, seq)` | Like `partition` but includes the incomplete final tuple. |
| `peek` | `peek(seq)` | Retrieve the first element and return `(first, iterator_with_first_restored)`. |
| `pluck` | `pluck(ind, seqs, default='__no__default__')` | Extract field(s) from each element. `pluck('name', records)`. |
| `random_sample` | `random_sample(prob, seq, random_state=None)` | Lazily sample elements with probability `prob`. |
| `reduceby` | `reduceby(key, binop, seq, init='__no__default__')` | Grouped reduction in a single pass. `reduceby(keyfunc, add, data)`. |
| `remove` | `remove(predicate, seq)` | Remove elements where predicate is True (opposite of `filter`). |
| `second` | `second(seq)` | Return the second element. |
| `sliding_window` | `sliding_window(n, seq)` | Sliding window of size `n`. `sliding_window(2, [1,2,3,4])` -> `(1,2),(2,3),(3,4)`. |
| `tail` | `tail(n, seq)` | Return the last `n` elements. |
| `take` | `take(n, seq)` | Return the first `n` elements as a list. |
| `take_nth` | `take_nth(n, seq)` | Take every nth element. `take_nth(2, [1,2,3,4,5])` -> `(1,3,5)`. |
| `topk` | `topk(k, seq, key=None)` | Return the `k` largest elements. Uses a heap, O(n log k). |
| `unique` | `unique(seq, key=None)` | Return unique elements, preserving order. |

### Functoolz

Higher-order functions for composing, currying, and transforming functions.

| Function | Signature | Description |
|----------|-----------|-------------|
| `apply` | `apply(*func_and_args, **kwargs)` | Call a function with positional and keyword arguments. `apply(f, x)` == `f(x)`. |
| `complement` | `complement(func)` | Logical negation of a predicate. `complement(iseven)(3)` -> `True`. |
| `compose` | `compose(*funcs)` | Right-to-left function composition. `compose(f, g)(x)` == `f(g(x))`. |
| `compose_left` | `compose_left(*funcs)` | Left-to-right function composition. `compose_left(f, g)(x)` == `g(f(x))`. |
| `curry` | `curry(func, *args, **kwargs)` | Automatic partial application. Returns a new function until all args are provided. |
| `do` | `do(func, x)` | Call `func(x)` for side effect, then return `x`. For inserting logging into pipelines. |
| `excepts` | `excepts(exc, func, handler=None)` | Catch exceptions functionally. Returns `handler(e)` on exception, `func(x)` otherwise. |
| `flip` | `flip(func)` | Reverse the argument order. `flip(func)(a, b)` == `func(b, a)`. |
| `identity` | `identity(x)` | Return `x` unchanged. Useful as default/no-op in pipelines. |
| `juxt` | `juxt(*funcs)` | Apply multiple functions to the same arguments. `juxt(f, g)(x)` -> `(f(x), g(x))`. |
| `memoize` | `memoize(func, cache=None, key=None)` | Cache function results. Supports custom cache backends. |
| `pipe` | `pipe(data, *funcs)` | Thread data through a sequence of functions. `pipe(x, f, g)` == `g(f(x))`. |
| `thread_first` | `thread_first(val, *forms)` | Thread value as the **first** argument through a sequence of function calls. |
| `thread_last` | `thread_last(val, *forms)` | Thread value as the **last** argument through a sequence of function calls. |

### Dicttoolz

Functions for working with dictionaries in a functional, non-mutating style.

| Function | Signature | Description |
|----------|-----------|-------------|
| `assoc` | `assoc(d, key, value, factory=dict)` | Return a new dict with `key` set to `value`. Does not mutate `d`. |
| `assoc_in` | `assoc_in(d, keys, value, factory=dict)` | Set a value in a nested dict. `assoc_in({}, ['a','b'], 1)` -> `{'a': {'b': 1}}`. |
| `dissoc` | `dissoc(d, *keys, factory=dict)` | Return a new dict without specified keys. |
| `get_in` | `get_in(keys, coll, default=None, no_default=False)` | Get a value from a nested dict. `get_in(['a','b'], {'a': {'b': 1}})` -> `1`. |
| `itemfilter` | `itemfilter(predicate, d, factory=dict)` | Filter dict by `(key, value)` pairs. |
| `itemmap` | `itemmap(func, d, factory=dict)` | Apply function to each `(key, value)` pair, returning new dict. |
| `keyfilter` | `keyfilter(predicate, d, factory=dict)` | Filter dict by keys. `keyfilter(lambda k: k != 'x', d)`. |
| `keymap` | `keymap(func, d, factory=dict)` | Apply function to each key. `keymap(str.upper, {'a': 1})` -> `{'A': 1}`. |
| `merge` | `merge(*dicts, **kwargs)` | Merge dictionaries left-to-right. Later values win. |
| `merge_with` | `merge_with(func, *dicts, **kwargs)` | Merge dicts, applying `func` to collisions. `merge_with(sum, d1, d2)`. |
| `update_in` | `update_in(d, keys, func, default=None, factory=dict)` | Update a value in a nested dict by applying a function. |
| `valfilter` | `valfilter(predicate, d, factory=dict)` | Filter dict by values. `valfilter(lambda v: v > 0, d)`. |
| `valmap` | `valmap(func, d, factory=dict)` | Apply function to each value. `valmap(len, {'a': [1,2], 'b': [3]})` -> `{'a': 2, 'b': 1}`. |

## Key Patterns

### `pipe()` for Data Pipeline Composition

`pipe` is the workhorse of toolz. It threads a value through a sequence of functions left-to-right, making data transformations read top-to-bottom like shell pipelines.

```python
from toolz import pipe, curry
from toolz.curried import map, filter

result = pipe(
    range(100),
    filter(lambda x: x % 2 == 0),   # keep evens
    map(lambda x: x ** 2),           # square them
    sum,                              # add them up
)
# result: 161700
```

The key insight: use `toolz.curried` for pre-curried versions of builtins that accept the iterable last, making them pipeline-friendly.

### `curry()` for Partial Application

`curry` automatically returns a new function when called with fewer arguments than expected. Unlike `functools.partial`, it chains automatically:

```python
from toolz import curry

@curry
def add(x, y):
    return x + y

@curry
def multiply(x, y):
    return x * y

add5 = add(5)
double = multiply(2)

add5(3)     # 8
double(10)  # 20

# Compose curried functions in pipelines
from toolz import pipe

pipe(10, add(5), multiply(3))  # (10 + 5) * 3 = 45
```

### `compose()` vs `compose_left()` vs `pipe()`

All three combine functions, but differ in argument order and usage:

```python
from toolz import compose, compose_left, pipe

def double(x): return x * 2
def inc(x): return x + 1
def square(x): return x ** 2

# compose: right-to-left (mathematical notation)
# Reads "square of inc of double of x"
f = compose(square, inc, double)
f(3)  # square(inc(double(3))) = square(inc(6)) = square(7) = 49

# compose_left: left-to-right (pipeline notation)
# Reads "double, then inc, then square"
g = compose_left(double, inc, square)
g(3)  # square(inc(double(3))) = 49  (same result!)

# pipe: like compose_left, but immediately applies to data
pipe(3, double, inc, square)  # 49
```

**When to use each:**
- `pipe(data, f, g, h)` -- you have data now and want to transform it
- `compose_left(f, g, h)` -- you want a reusable function, reading left-to-right
- `compose(h, g, f)` -- you want a reusable function, reading right-to-left (math convention)

### `groupby()` + `valmap()` for Data Transformation

This is the split-apply-combine pattern, done functionally:

```python
from toolz import groupby, valmap, pipe

records = [
    {"name": "Alice", "dept": "Engineering", "salary": 120_000},
    {"name": "Bob", "dept": "Engineering", "salary": 110_000},
    {"name": "Carol", "dept": "Marketing", "salary": 95_000},
    {"name": "Dave", "dept": "Marketing", "salary": 90_000},
    {"name": "Eve", "dept": "Engineering", "salary": 130_000},
]

# Average salary by department
avg_by_dept = pipe(
    records,
    lambda rs: groupby("dept", rs),                        # split
    lambda groups: valmap(lambda g: [r["salary"] for r in g], groups),  # extract
    lambda salaries: valmap(lambda s: sum(s) / len(s), salaries),       # combine
)
# {'Engineering': 120000.0, 'Marketing': 92500.0}
```

A cleaner version using curried functions:

```python
from toolz import pipe, groupby, valmap, pluck

def mean(xs):
    xs = list(xs)
    return sum(xs) / len(xs)

avg_by_dept = pipe(
    records,
    groupby("dept"),
    valmap(lambda g: mean(pluck("salary", g))),
)
```

### `memoize()` for Caching

`memoize` caches function results based on arguments. Simpler than `functools.lru_cache` for the common case:

```python
from toolz import memoize

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)  # instant, no stack overflow thanks to bottom-up caching

# Custom cache backend (e.g., LRU with max size)
from functools import lru_cache

@memoize(cache={})
def expensive_lookup(key):
    ...

# Inspect or clear cache
expensive_lookup.cache  # the underlying dict
```

### `excepts()` for Functional Error Handling

`excepts` wraps a function to catch specific exceptions, returning a handler's result instead of raising:

```python
from toolz import excepts, pipe
from toolz.curried import map

safe_int = excepts(ValueError, int, lambda e: None)

safe_int("42")    # 42
safe_int("hello") # None
safe_int("")      # None

# Use in pipelines to handle bad data gracefully
raw = ["1", "2", "bad", "4", "oops", "6"]

pipe(
    raw,
    map(safe_int),         # [1, 2, None, 4, None, 6]
    filter(None.__ne__),   # [1, 2, 4, 6]
    list,
)
# [1, 2, 4, 6]
```

Handle multiple exception types:

```python
safe_divide = excepts(
    (ZeroDivisionError, TypeError),
    lambda x: 100 / x,
    lambda e: float("inf"),
)

safe_divide(5)    # 20.0
safe_divide(0)    # inf
safe_divide(None) # inf
```

## cytoolz -- Cython-Accelerated Version

`cytoolz` is a Cython implementation of toolz with an identical API. It provides significant speedups for hot paths.

### When to Use cytoolz

| Scenario | Recommendation |
|----------|----------------|
| General scripting | `toolz` is fine, zero compile overhead |
| Data pipelines with millions of records | `cytoolz` gives 2-5x speedup on function call overhead |
| Libraries consumed by others | Use `toolz`, let consumers choose `cytoolz` |
| Performance-critical inner loops | `cytoolz` -- the function call overhead reduction matters |

### Typical Speedups

- `curry` dispatch: ~3-5x faster
- `pipe` with many stages: ~2-4x faster
- `groupby`, `frequencies`, `reduceby`: ~2-3x faster
- `compose` call overhead: ~3-5x faster

### Installation and Usage

```bash
pip install cytoolz
```

```python
# Direct import
from cytoolz import pipe, curry, groupby

# Graceful fallback pattern (recommended for libraries)
try:
    from cytoolz import pipe, curry, groupby
    from cytoolz.curried import map, filter
except ImportError:
    from toolz import pipe, curry, groupby
    from toolz.curried import map, filter
```

`cytoolz` requires a C compiler to build from source, but binary wheels are available for most platforms. If you cannot install `cytoolz` (e.g., in constrained environments), `toolz` works identically, just slower on function dispatch overhead.

### cytoolz.curried

Like `toolz.curried`, `cytoolz.curried` provides pre-curried versions of all functions:

```python
from cytoolz.curried import get, pluck, groupby, valmap
```

## `toolz.curried` Module Reference

The `toolz.curried` namespace provides curried versions of all toolz functions, plus curried versions of key builtins. This is essential for building clean pipelines.

```python
import toolz.curried as tz

# All toolz functions are curried:
tz.map(str.upper)            # returns a curried function waiting for an iterable
tz.filter(bool)              # returns a curried function waiting for an iterable
tz.get(0)                    # returns a curried function waiting for a sequence
tz.pluck("name")             # returns a curried function waiting for a sequence of dicts
tz.groupby(len)              # returns a curried function waiting for a sequence
tz.keymap(str.upper)         # returns a curried function waiting for a dict
tz.valmap(len)               # returns a curried function waiting for a dict
tz.valfilter(bool)           # returns a curried function waiting for a dict

# Complete pipeline using only curried functions:
result = tz.pipe(
    [{"name": "Alice", "score": 90}, {"name": "Bob", "score": 85}],
    tz.pluck("score"),
    tz.map(lambda s: s * 1.1),
    sum,
)
```
