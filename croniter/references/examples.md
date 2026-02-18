# croniter — Examples & Gotchas

> Part of the croniter skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Day-of-Month vs Day-of-Week: OR vs AND](#1-day-of-month-vs-day-of-week-or-vs-and)
  - [2. The Iterator Is Stateful](#2-the-iterator-is-stateful)
  - [3. 6-field vs 5-field Detection](#3-6-field-vs-5-field-detection)
  - [4. `get_next()` Never Returns the Start Time](#4-get_next-never-returns-the-start-time)
  - [5. Timezone-Aware Datetimes](#5-timezone-aware-datetimes)
  - [6. Month Lengths](#6-month-lengths)
- [Complete Examples](#complete-examples)
  - [Example 1: Next N Occurrences of a Schedule](#example-1-next-n-occurrences-of-a-schedule)
  - [Example 2: Time Until Next Run](#example-2-time-until-next-run)
  - [Example 3: Validate User-Supplied Cron Expressions](#example-3-validate-user-supplied-cron-expressions)
  - [Example 4: Check if a Datetime Matches a Schedule](#example-4-check-if-a-datetime-matches-a-schedule)
  - [Example 5: Using Seconds (6-field)](#example-5-using-seconds-6-field)
  - [Example 6: Iterating Backwards](#example-6-iterating-backwards)
- [See Also](#see-also)

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
