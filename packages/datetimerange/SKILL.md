---
name: datetimerange
description: Datetime range operations including intersection, union, containment, and iteration. Use when working with time intervals, checking schedule overlaps, computing availability windows, or iterating over time ranges. Triggers on datetime range, time interval, datetimerange, schedule overlap, time intersection, availability.
---

# DateTimeRange — Time Interval Operations (v2.2.1)

## Quick Start

```bash
pip install DateTimeRange
```

```python
from datetimerange import DateTimeRange

r = DateTimeRange("2025-01-01T09:00:00", "2025-01-01T17:00:00")
"2025-01-01T12:00:00" in r  # True (containment check)
```

## Key Patterns

### Create and check containment
```python
from datetimerange import DateTimeRange
from datetime import datetime

r = DateTimeRange("2025-01-01T09:00:00", "2025-01-01T17:00:00")
datetime(2025, 1, 1, 12) in r    # True
```

### Intersection and overlap
```python
r1 = DateTimeRange("2025-01-01T09:00:00", "2025-01-01T12:00:00")
r2 = DateTimeRange("2025-01-01T11:00:00", "2025-01-01T14:00:00")

r1.is_intersection(r2)  # True (they overlap)
r1.intersection(r2)     # DateTimeRange 11:00-12:00
```

### Iteration
```python
from datetime import timedelta
for dt in r.range(timedelta(hours=1)):
    print(dt)
```

## References

- **[api.md](references/api.md)** — Constructor, properties, containment, overlap detection, intersection, encompass, truncation, iteration, comparison, and subtraction APIs
- **[examples.md](references/examples.md)** — Common gotchas (in-place mutation, no-overlap errors, timezone mixing), meeting overlap detection, business hours, time slot splitting, availability finding
