# DateTimeRange — Examples & Gotchas

> Part of the datetimerange skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [truncate() Modifies In Place](#1-truncate-modifies-in-place)
  - [intersection() Raises on No Overlap](#2-intersection-raises-on-no-overlap)
  - [String Parsing Depends on dateutil](#3-string-parsing-depends-on-dateutil)
  - [Start Must Be Before End](#4-start-must-be-before-end)
  - [Timezone Consistency](#5-timezone-consistency)
- [Complete Examples](#complete-examples)
  - [Meeting Overlap Detection](#example-1-meeting-overlap-detection)
  - [Business Hours Check](#example-2-business-hours-check)
  - [Splitting a Day into Hourly Slots](#example-3-splitting-a-day-into-hourly-slots)
  - [Finding Available Time](#example-4-finding-available-time)
  - [Merging Overlapping Ranges](#example-5-merging-overlapping-ranges)
- [See Also](#see-also)

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
