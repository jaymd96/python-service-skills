---
name: pendulum
description: Timezone-aware datetime replacement with immutable, fluent API. Use when working with dates/times, timezone conversions, date arithmetic, human-readable time diffs, or replacing stdlib datetime. Triggers on datetime, pendulum, timezone, date arithmetic, time parsing, human readable dates.
---

# pendulum — Timezone-Safe Datetime (v3.0.0)

## Quick Start

```bash
pip install pendulum
```

```python
import pendulum

now = pendulum.now("Europe/London")  # always timezone-aware
tomorrow = now.add(days=1)
yesterday = now.subtract(days=1)
now.diff_for_humans()  # "just now"
```

## Key Patterns

### Creation
```python
pendulum.now("UTC")
pendulum.datetime(2024, 1, 15, tz="Europe/London")
pendulum.parse("2024-01-15T10:30:00+00:00")
pendulum.from_format("15/01/2024", "DD/MM/YYYY")
```

### Arithmetic (immutable — returns new objects)
```python
dt = pendulum.now()
new_dt = dt.add(hours=2, minutes=30)  # dt is unchanged
dt.subtract(weeks=1)
dt.start_of("week")   # Monday 00:00
dt.end_of("month")    # Last day 23:59:59.999999
```

### Diffs and periods
```python
diff = dt1.diff(dt2)
diff.in_hours()          # total hours
dt1.diff_for_humans(dt2) # "2 hours before"
period = pendulum.period(start, end)
for dt in period.range("days"):
    print(dt)
```

## References

- **[api.md](references/api.md)** — Complete API for creation, arithmetic, formatting, timezone handling, durations, periods, and format tokens
- **[examples.md](references/examples.md)** — Gotchas (2.x vs 3.x, JSON, SQLAlchemy, immutability), and complete code examples
