# croniter — API Reference

> Part of the croniter skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [`croniter(expr, start_time, ...)`](#croniterexpr-start_time-)
- [`.get_next(ret_type=None)`](#get_nextret_typenone)
- [`.get_prev(ret_type=None)`](#get_prevret_typenone)
- [`.get_current(ret_type=None)`](#get_currentret_typenone)
- [`.all_next(ret_type=None)` / `.all_prev(ret_type=None)`](#all_nextret_typenone--all_prevret_typenone)
- [`croniter.match(expr, dt)`](#cronitermatchexpr-dt)
- [`croniter.is_valid(expr)`](#croniteris_validexpr)
- [Cron Expression Syntax](#cron-expression-syntax)
  - [Standard 5-Field Format](#standard-5-field-format)
  - [6-Field Format (with Seconds)](#6-field-format-with-seconds)
  - [7-Field Format (with Seconds and Year)](#7-field-format-with-seconds-and-year)
  - [Field Operators](#field-operators)
  - [Special Expressions](#special-expressions)
  - [Hash Expressions (for Jitter)](#hash-expressions-for-jitter)

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
