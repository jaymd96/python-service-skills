# DateTimeRange — API Reference

> Part of the datetimerange skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API Reference](#core-api-reference)
  - [DateTimeRange(start, end)](#datetimerangestart-end)
  - [Properties](#properties)
  - [is_set()](#is_set----check-if-range-is-defined)
  - [set_time_range(start, end)](#set_time_rangestart-end)
- [Containment Checks](#containment-checks)
- [Overlap Detection](#overlap-detection)
  - [is_intersection(other)](#is_intersectionother----check-for-overlap)
  - [intersection(other)](#intersectionother----get-overlapping-range)
  - [encompass(other)](#encompassother----get-union-bounding-range)
- [Truncation](#truncation)
- [Iteration](#iteration)
- [Comparison Operators](#comparison-operators)
- [Subtraction](#subtraction)

## Core API Reference

### `DateTimeRange(start, end)`

Create a datetime range from two datetime-like objects.

```python
from datetimerange import DateTimeRange
from datetime import datetime

# From datetime objects
r = DateTimeRange(
    datetime(2025, 1, 1, 0, 0),
    datetime(2025, 1, 31, 23, 59),
)
print(r)  # 2025-01-01T00:00:00 - 2025-01-31T23:59:00

# From strings (auto-parsed)
r = DateTimeRange("2025-01-01T00:00:00", "2025-01-31T23:59:00")
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `start_datetime` | `datetime` | Start of the range |
| `end_datetime` | `datetime` | End of the range |
| `timedelta` | `timedelta` | Duration of the range (`end - start`) |

```python
r = DateTimeRange("2025-01-01T09:00:00", "2025-01-01T17:00:00")
r.start_datetime  # datetime(2025, 1, 1, 9, 0)
r.end_datetime    # datetime(2025, 1, 1, 17, 0)
r.timedelta       # timedelta(hours=8)
```

### `is_set()` -- Check if Range Is Defined

Returns `True` if both start and end are set (not `None`).

```python
r = DateTimeRange()
r.is_set()  # False

r = DateTimeRange("2025-01-01", "2025-01-31")
r.is_set()  # True
```

### `set_time_range(start, end)`

Set or replace the start and end times.

```python
r = DateTimeRange()
r.set_time_range("2025-06-01T00:00:00", "2025-06-30T23:59:59")
r.is_set()  # True
```

---

## Containment Checks

### `__contains__` -- `in` operator

Check whether a datetime or another `DateTimeRange` is within the range.

```python
from datetimerange import DateTimeRange
from datetime import datetime

r = DateTimeRange("2025-01-01T00:00:00", "2025-01-31T23:59:59")

# Check datetime containment
datetime(2025, 1, 15) in r     # True
datetime(2025, 2, 15) in r     # False
"2025-01-15T12:00:00" in r     # True (string also works)

# Check range containment (subset)
sub = DateTimeRange("2025-01-10", "2025-01-20")
sub in r  # True

outer = DateTimeRange("2024-12-01", "2025-02-28")
outer in r  # False (outer is larger)
```

---

## Overlap Detection

### `is_intersection(other)` -- Check for Overlap

```python
r1 = DateTimeRange("2025-01-01", "2025-01-15")
r2 = DateTimeRange("2025-01-10", "2025-01-25")
r3 = DateTimeRange("2025-02-01", "2025-02-15")

r1.is_intersection(r2)  # True (they overlap)
r1.is_intersection(r3)  # False (no overlap)
```

### `intersection(other)` -- Get Overlapping Range

Returns a new `DateTimeRange` representing the intersection, or raises `ValueError` if no overlap.

```python
r1 = DateTimeRange("2025-01-01", "2025-01-15")
r2 = DateTimeRange("2025-01-10", "2025-01-25")

overlap = r1.intersection(r2)
print(overlap)  # 2025-01-10T00:00:00 - 2025-01-15T00:00:00
```

### `encompass(other)` -- Get Union (Bounding Range)

Returns the smallest `DateTimeRange` that contains both ranges.

```python
r1 = DateTimeRange("2025-01-01", "2025-01-15")
r2 = DateTimeRange("2025-01-10", "2025-01-25")

merged = r1.encompass(r2)
print(merged)  # 2025-01-01T00:00:00 - 2025-01-25T00:00:00
```

---

## Truncation

### `truncate(percentage)`

Shrink the range by a percentage from both ends.

```python
r = DateTimeRange("2025-01-01T00:00:00", "2025-01-01T10:00:00")
r.truncate(10)  # Remove 10% from each end
print(r)  # 2025-01-01T01:00:00 - 2025-01-01T09:00:00
# (1 hour removed from each end of a 10-hour range)
```

**Note:** `truncate()` modifies the range **in place** and returns `None`.

---

## Iteration

### `range(step)`

Iterate over the range at a given step interval.

```python
from datetimerange import DateTimeRange
from datetime import timedelta

r = DateTimeRange("2025-01-01T00:00:00", "2025-01-01T06:00:00")

for dt in r.range(timedelta(hours=2)):
    print(dt)
# 2025-01-01 00:00:00
# 2025-01-01 02:00:00
# 2025-01-01 04:00:00
# 2025-01-01 06:00:00
```

---

## Comparison Operators

`DateTimeRange` supports equality and comparison operators:

```python
r1 = DateTimeRange("2025-01-01", "2025-01-10")
r2 = DateTimeRange("2025-01-01", "2025-01-10")
r3 = DateTimeRange("2025-01-01", "2025-01-15")

r1 == r2  # True (same start and end)
r1 != r3  # True

# Ordering is based on start_datetime, then end_datetime
r1 < r3   # True (same start, but r1 ends earlier)
```

---

## Subtraction

### `subtract(other)` -- Remove Overlap

Returns a list of `DateTimeRange` objects representing the parts of `self` that do not overlap with `other`.

```python
r1 = DateTimeRange("2025-01-01", "2025-01-31")
r2 = DateTimeRange("2025-01-10", "2025-01-20")

result = r1.subtract(r2)
for r in result:
    print(r)
# 2025-01-01T00:00:00 - 2025-01-10T00:00:00
# 2025-01-20T00:00:00 - 2025-01-31T00:00:00
```
