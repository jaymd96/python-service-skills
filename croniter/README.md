# croniter

A Python library for **parsing cron expressions** and iterating over scheduled times.

`croniter` takes a cron expression string and a starting datetime, then generates an infinite sequence of future (or past) datetimes matching that schedule. It supports the standard 5-field cron syntax, extended 6-field (with seconds) and 7-field (with year) syntax, special expressions like `@daily`, hash expressions for jitter, and a `match()` function for testing whether a datetime matches a cron expression.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `croniter` |
| **Latest version** | 6.0.2 (2025) |
| **Python support** | Python >= 3.7 |
| **License** | MIT |
| **Repository** | https://github.com/kiorky/croniter |
| **Documentation** | https://github.com/kiorky/croniter#readme |

---

## Installation

```bash
pip install croniter
```

`croniter` depends on `python-dateutil`. It will be installed automatically.

```python
from croniter import croniter
from datetime import datetime
```

---

## Core API Reference

### `croniter(expr, start_time=None, ret_type=datetime, day_or=True, max_years_between_matches=50)`

Create an iterator for a cron expression.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `expr` | `str` | (required) | Cron expression |
| `start_time` | `datetime`, `float`, or `None` | `None` (now) | Starting point for iteration |
| `ret_type` | `type` | `datetime` | Return type: `datetime`, `float`, or `str` |
| `day_or` | `bool` | `True` | If `True`, day-of-month OR day-of-week; if `False`, AND |
| `max_years_between_matches` | `int` | `50` | Max years to search forward/backward for a match |

```python
from croniter import croniter
from datetime import datetime

base = datetime(2025, 1, 1, 0, 0)
cron = croniter("0 9 * * MON-FRI", base)  # 9 AM weekdays
```

### `.get_next(ret_type=None)`

Get the next matching datetime after the current position.

```python
cron = croniter("*/15 * * * *", datetime(2025, 1, 1, 0, 0))
cron.get_next(datetime)  # datetime(2025, 1, 1, 0, 15)
cron.get_next(datetime)  # datetime(2025, 1, 1, 0, 30)
cron.get_next(datetime)  # datetime(2025, 1, 1, 0, 45)
```

### `.get_prev(ret_type=None)`

Get the previous matching datetime before the current position.

```python
cron = croniter("0 * * * *", datetime(2025, 6, 15, 12, 30))
cron.get_prev(datetime)  # datetime(2025, 6, 15, 12, 0)
cron.get_prev(datetime)  # datetime(2025, 6, 15, 11, 0)
```

### `.get_current(ret_type=None)`

Get the current position of the iterator without advancing it.

```python
cron = croniter("0 0 * * *", datetime(2025, 1, 1))
cron.get_next(datetime)       # datetime(2025, 1, 2, 0, 0)
cron.get_current(datetime)    # datetime(2025, 1, 2, 0, 0) -- same, no advance
```

### `.all_next(ret_type=None)` / `.all_prev(ret_type=None)`

Return infinite generators yielding all subsequent/previous matches.

```python
cron = croniter("0 0 1 * *", datetime(2025, 1, 1))  # midnight on the 1st

# Get the next 5 monthly occurrences
import itertools
next_five = list(itertools.islice(cron.all_next(datetime), 5))
# [datetime(2025, 2, 1), datetime(2025, 3, 1), datetime(2025, 4, 1),
#  datetime(2025, 5, 1), datetime(2025, 6, 1)]
```

**Warning:** These are infinite generators. Always use `itertools.islice()` or a break condition.

### `croniter.match(expr, dt)`  (static method)

Test whether a specific datetime matches a cron expression.

```python
from croniter import croniter
from datetime import datetime

dt = datetime(2025, 3, 15, 9, 0)  # Saturday 9:00 AM

croniter.match("0 9 * * SAT", dt)    # True
croniter.match("0 9 * * MON-FRI", dt)  # False
croniter.match("*/5 * * * *", dt)    # True (minute 0 matches */5)
```

### `croniter.is_valid(expr)`  (static method)

Check whether a cron expression string is syntactically valid.

```python
croniter.is_valid("0 9 * * MON")      # True
croniter.is_valid("*/15 * * * *")      # True
croniter.is_valid("not a cron")        # False
croniter.is_valid("60 * * * *")        # False (minute 60 out of range)
```

---

## Cron Expression Syntax

### Standard 5-Field Format

```
 ┌───────────── minute (0-59)
 │ ┌───────────── hour (0-23)
 │ │ ┌───────────── day of month (1-31)
 │ │ │ ┌───────────── month (1-12 or JAN-DEC)
 │ │ │ │ ┌───────────── day of week (0-6, SUN=0 or MON-SUN)
 │ │ │ │ │
 * * * * *
```

### 6-Field Format (with Seconds)

```
 ┌───────────── second (0-59)
 │ ┌───────────── minute (0-59)
 │ │ ┌───────────── hour (0-23)
 │ │ │ ┌───────────── day of month (1-31)
 │ │ │ │ ┌───────────── month (1-12)
 │ │ │ │ │ ┌───────────── day of week (0-6)
 │ │ │ │ │ │
 * * * * * *
```

### 7-Field Format (with Seconds and Year)

```
 ┌───────────── second (0-59)
 │ ┌───────────── minute (0-59)
 │ │ ┌───────────── hour (0-23)
 │ │ │ ┌───────────── day of month (1-31)
 │ │ │ │ ┌───────────── month (1-12)
 │ │ │ │ │ ┌───────────── day of week (0-6)
 │ │ │ │ │ │ ┌───────────── year (1970-2099)
 │ │ │ │ │ │ │
 * * * * * * *
```

### Field Operators

| Operator | Example | Meaning |
|----------|---------|---------|
| `*` | `* * * * *` | Every unit |
| `,` | `1,15 * * * *` | List (minute 1 and 15) |
| `-` | `1-5 * * * *` | Range (minutes 1 through 5) |
| `/` | `*/10 * * * *` | Step (every 10 minutes) |
| `L` | `0 0 L * *` | Last day of month |
| `W` | `0 0 15W * *` | Nearest weekday to the 15th |
| `#` | `0 0 * * 5#3` | Third Friday of the month |

### Special Expressions

| Expression | Equivalent |
|------------|------------|
| `@yearly` / `@annually` | `0 0 1 1 *` |
| `@monthly` | `0 0 1 * *` |
| `@weekly` | `0 0 * * 0` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@hourly` | `0 * * * *` |

### Hash Expressions (for Jitter)

Hash expressions distribute executions across time slots to avoid thundering-herd problems. They use a hash of a seed value to deterministically pick a slot:

```python
# H means "pick a consistent value based on hash of the seed"
cron = croniter("H H * * *", base, hash_id="my-job-name")
cron.get_next(datetime)  # Deterministic time based on hash of "my-job-name"
```

The `H` token can be used in place of any numeric field. `H(0-30)` restricts the hash to a range.

---

## Gotchas and Common Mistakes

### 1. Day-of-Month vs Day-of-Week: OR vs AND

By default (`day_or=True`), if both day-of-month and day-of-week are specified, the match is an OR: either field matching is sufficient. Set `day_or=False` for AND behavior:

```python
# OR (default): fires on the 1st OR on Mondays
croniter("0 0 1 * MON", base, day_or=True)

# AND: fires only on the 1st IF it's also a Monday
croniter("0 0 1 * MON", base, day_or=False)
```

### 2. The Iterator Is Stateful

Calling `get_next()` advances the iterator. Calling it again gets the *next* occurrence, not the same one. Use `get_current()` to re-read the position.

### 3. 6-field vs 5-field Detection

`croniter` auto-detects whether an expression has 5, 6, or 7 fields. If you accidentally pass a 6-field expression expecting 5-field behavior, the first field will be interpreted as seconds.

### 4. `get_next()` Never Returns the Start Time

If `start_time` exactly matches the cron expression, `get_next()` will still advance to the **next** occurrence. Use `match()` to check the start time itself.

```python
base = datetime(2025, 1, 1, 0, 0)  # midnight
cron = croniter("0 0 * * *", base)  # daily at midnight
cron.get_next(datetime)  # 2025-01-02 00:00:00, NOT 2025-01-01
```

### 5. Timezone-Aware Datetimes

If you pass a timezone-aware datetime as `start_time`, the returned datetimes will also be timezone-aware. Mix-ups between naive and aware datetimes will raise errors.

### 6. Month Lengths

Expressions like `0 0 31 * *` will skip months that have fewer than 31 days. This is correct behavior but can be surprising -- it will fire on Jan 31, Mar 31, May 31, etc.

---

## Complete Examples

### Example 1: Next N Occurrences of a Schedule

```python
from croniter import croniter
from datetime import datetime
import itertools

base = datetime(2025, 6, 1, 0, 0)
cron = croniter("30 8 * * MON-FRI", base)  # 8:30 AM on weekdays

next_10 = list(itertools.islice(cron.all_next(datetime), 10))
for dt in next_10:
    print(dt.strftime("%Y-%m-%d %H:%M (%A)"))
# 2025-06-02 08:30 (Monday)
# 2025-06-03 08:30 (Tuesday)
# ...
```

### Example 2: Time Until Next Run

```python
from croniter import croniter
from datetime import datetime

now = datetime.now()
cron = croniter("0 */2 * * *", now)  # every 2 hours
next_run = cron.get_next(datetime)
delta = next_run - now

print(f"Next run: {next_run}")
print(f"Time until next run: {delta}")
```

### Example 3: Validate User-Supplied Cron Expressions

```python
from croniter import croniter

def validate_cron(expr: str) -> tuple[bool, str]:
    if croniter.is_valid(expr):
        return True, "Valid cron expression"
    else:
        return False, f"Invalid cron expression: {expr!r}"

print(validate_cron("0 9 * * 1-5"))     # (True, 'Valid cron expression')
print(validate_cron("*/5 * * * *"))      # (True, 'Valid cron expression')
print(validate_cron("bad expression"))   # (False, "Invalid cron expression: 'bad expression'")
```

### Example 4: Check if a Datetime Matches a Schedule

```python
from croniter import croniter
from datetime import datetime

schedule = "0 9 * * MON"

test_dates = [
    datetime(2025, 3, 10, 9, 0),   # Monday 9:00 AM
    datetime(2025, 3, 10, 10, 0),  # Monday 10:00 AM
    datetime(2025, 3, 11, 9, 0),   # Tuesday 9:00 AM
]

for dt in test_dates:
    matches = croniter.match(schedule, dt)
    print(f"{dt} -> {'MATCH' if matches else 'no match'}")
# 2025-03-10 09:00:00 -> MATCH
# 2025-03-10 10:00:00 -> no match
# 2025-03-11 09:00:00 -> no match
```

### Example 5: Using Seconds (6-field)

```python
from croniter import croniter
from datetime import datetime

base = datetime(2025, 1, 1, 0, 0, 0)
cron = croniter("*/30 * * * * *", base)  # every 30 seconds

for _ in range(5):
    print(cron.get_next(datetime))
# 2025-01-01 00:00:30
# 2025-01-01 00:01:00
# 2025-01-01 00:01:30
# 2025-01-01 00:02:00
# 2025-01-01 00:02:30
```

### Example 6: Iterating Backwards

```python
from croniter import croniter
from datetime import datetime
import itertools

now = datetime(2025, 6, 15, 12, 0)
cron = croniter("0 0 * * *", now)  # daily at midnight

last_5 = list(itertools.islice(cron.all_prev(datetime), 5))
for dt in last_5:
    print(dt)
# 2025-06-15 00:00:00
# 2025-06-14 00:00:00
# 2025-06-13 00:00:00
# 2025-06-12 00:00:00
# 2025-06-11 00:00:00
```

---

## See Also

- [croniter on GitHub](https://github.com/kiorky/croniter)
- [Cron Expression Reference (crontab.guru)](https://crontab.guru/)
- [python-dateutil](https://dateutil.readthedocs.io/) -- required dependency
