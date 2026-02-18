# DateTimeRange

A Python library for **datetime range operations** -- intersection, union, containment, truncation, iteration, and more.

`DateTimeRange` provides a single class that represents a range between two datetimes and offers rich operations for working with time intervals. It supports overlap detection, intersection, union, truncation, iteration at arbitrary steps, and containment checks for both datetimes and other ranges. Useful for scheduling, availability calculations, time-series processing, and calendar logic.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `DateTimeRange` |
| **Latest version** | 2.2.1 (2024) |
| **Python support** | Python >= 3.7 |
| **License** | MIT |
| **Repository** | https://github.com/thombashi/DateTimeRange |
| **Documentation** | https://datetimerange.readthedocs.io |

The import name is `datetimerange`.

---

## Installation

```bash
pip install DateTimeRange
```

```python
from datetimerange import DateTimeRange
from datetime import datetime
```

---

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

---

## Gotchas and Common Mistakes

### 1. `truncate()` Modifies In Place

Unlike most Python APIs that return new objects, `truncate()` modifies the `DateTimeRange` in place:

```python
r = DateTimeRange("2025-01-01", "2025-01-31")
result = r.truncate(10)  # result is None!
# r itself has been modified
```

### 2. `intersection()` Raises on No Overlap

If two ranges do not overlap, `intersection()` raises a `ValueError`. Always check with `is_intersection()` first:

```python
r1 = DateTimeRange("2025-01-01", "2025-01-10")
r2 = DateTimeRange("2025-02-01", "2025-02-10")

if r1.is_intersection(r2):
    overlap = r1.intersection(r2)
else:
    print("No overlap")
```

### 3. String Parsing Depends on `dateutil`

When you pass strings to the constructor, they are parsed by `dateutil.parser.parse()`. Ambiguous date strings (like `"01/02/03"`) may be interpreted differently than expected. Use ISO 8601 format (`"2025-01-15T09:00:00"`) for unambiguous dates.

### 4. Start Must Be Before End

Creating a range where start > end will result in a negative `timedelta` and may cause unexpected behavior in operations:

```python
r = DateTimeRange("2025-01-31", "2025-01-01")
r.timedelta  # timedelta(days=-30) -- negative!
```

Validate your inputs to ensure `start <= end`.

### 5. Timezone Consistency

Mixing timezone-aware and timezone-naive datetimes within a range or across ranges in operations will raise `TypeError`. Keep all datetimes consistently naive or consistently aware.

---

## Complete Examples

### Example 1: Meeting Overlap Detection

```python
from datetimerange import DateTimeRange

meetings = [
    DateTimeRange("2025-03-15T09:00:00", "2025-03-15T10:30:00"),
    DateTimeRange("2025-03-15T10:00:00", "2025-03-15T11:00:00"),
    DateTimeRange("2025-03-15T13:00:00", "2025-03-15T14:00:00"),
    DateTimeRange("2025-03-15T13:30:00", "2025-03-15T15:00:00"),
]

conflicts = []
for i in range(len(meetings)):
    for j in range(i + 1, len(meetings)):
        if meetings[i].is_intersection(meetings[j]):
            overlap = meetings[i].intersection(meetings[j])
            conflicts.append((i, j, overlap))

for i, j, overlap in conflicts:
    print(f"Meeting {i} and {j} overlap during: {overlap}")
# Meeting 0 and 1 overlap during: 2025-03-15T10:00:00 - 2025-03-15T10:30:00
# Meeting 2 and 3 overlap during: 2025-03-15T13:30:00 - 2025-03-15T14:00:00
```

### Example 2: Business Hours Check

```python
from datetimerange import DateTimeRange
from datetime import datetime

business_hours = DateTimeRange("2025-03-15T09:00:00", "2025-03-15T17:00:00")

events = [
    ("Standup", datetime(2025, 3, 15, 9, 30)),
    ("Early bird", datetime(2025, 3, 15, 7, 0)),
    ("Night owl", datetime(2025, 3, 15, 22, 0)),
    ("Lunch", datetime(2025, 3, 15, 12, 0)),
]

for name, dt in events:
    status = "during business hours" if dt in business_hours else "outside business hours"
    print(f"{name} ({dt.strftime('%H:%M')}): {status}")
# Standup (09:30): during business hours
# Early bird (07:00): outside business hours
# Night owl (22:00): outside business hours
# Lunch (12:00): during business hours
```

### Example 3: Splitting a Day into Hourly Slots

```python
from datetimerange import DateTimeRange
from datetime import timedelta

workday = DateTimeRange("2025-03-15T09:00:00", "2025-03-15T17:00:00")

slots = list(workday.range(timedelta(hours=1)))
for i in range(len(slots) - 1):
    slot = DateTimeRange(slots[i], slots[i + 1])
    print(f"Slot: {slot}")
# Slot: 2025-03-15T09:00:00 - 2025-03-15T10:00:00
# Slot: 2025-03-15T10:00:00 - 2025-03-15T11:00:00
# ... (8 one-hour slots)
```

### Example 4: Finding Available Time

```python
from datetimerange import DateTimeRange

full_day = DateTimeRange("2025-03-15T09:00:00", "2025-03-15T17:00:00")

busy = [
    DateTimeRange("2025-03-15T09:00:00", "2025-03-15T10:00:00"),
    DateTimeRange("2025-03-15T11:30:00", "2025-03-15T13:00:00"),
    DateTimeRange("2025-03-15T15:00:00", "2025-03-15T16:00:00"),
]

# Subtract busy periods to find free time
free = [full_day]
for b in busy:
    new_free = []
    for f in free:
        if f.is_intersection(b):
            new_free.extend(f.subtract(b))
        else:
            new_free.append(f)
    free = new_free

print("Available slots:")
for f in free:
    print(f"  {f}")
# Available slots:
#   2025-03-15T10:00:00 - 2025-03-15T11:30:00
#   2025-03-15T13:00:00 - 2025-03-15T15:00:00
#   2025-03-15T16:00:00 - 2025-03-15T17:00:00
```

### Example 5: Merging Overlapping Ranges

```python
from datetimerange import DateTimeRange

ranges = [
    DateTimeRange("2025-03-15T09:00:00", "2025-03-15T11:00:00"),
    DateTimeRange("2025-03-15T10:00:00", "2025-03-15T12:00:00"),
    DateTimeRange("2025-03-15T14:00:00", "2025-03-15T16:00:00"),
    DateTimeRange("2025-03-15T15:00:00", "2025-03-15T17:00:00"),
]

# Sort by start time
ranges.sort(key=lambda r: r.start_datetime)

merged = [ranges[0]]
for r in ranges[1:]:
    if merged[-1].is_intersection(r):
        merged[-1] = merged[-1].encompass(r)
    else:
        merged.append(r)

for m in merged:
    print(m)
# 2025-03-15T09:00:00 - 2025-03-15T12:00:00
# 2025-03-15T14:00:00 - 2025-03-15T17:00:00
```

---

## See Also

- [DateTimeRange on GitHub](https://github.com/thombashi/DateTimeRange)
- [DateTimeRange on PyPI](https://pypi.org/project/DateTimeRange/)
- [Documentation](https://datetimerange.readthedocs.io)
- [python-dateutil](https://dateutil.readthedocs.io/) -- used internally for date parsing
