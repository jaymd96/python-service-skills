# python-dateutil — API Reference

> Part of the python-dateutil skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [`parser.parse()` -- Flexible Date Parsing](#parserparse----flexible-date-parsing)
  - [Key Parameters](#key-parameters)
- [`relativedelta` -- Date Arithmetic](#relativedelta----date-arithmetic)
  - [Relative vs Absolute Arguments](#relative-vs-absolute-arguments)
- [`rrule` -- Recurrence Rules](#rrule----recurrence-rules)
  - [`rruleset` -- Combining Rules](#rruleset----combining-rules)
  - [`rrulestr` -- Parse iCalendar RRULE Strings](#rrulestr----parse-icalendar-rrule-strings)
- [`tz` -- Timezone Utilities](#tz----timezone-utilities)
- [`easter` -- Easter Date Calculation](#easter----easter-date-calculation)

### `parser.parse()` -- Flexible Date Parsing

Parses almost any human-readable date string into a `datetime` object.

```python
from dateutil import parser

parser.parse("2025-03-15")                    # datetime(2025, 3, 15, 0, 0)
parser.parse("March 15, 2025")                # datetime(2025, 3, 15, 0, 0)
parser.parse("15/03/2025")                    # datetime(2025, 3, 15, 0, 0)
parser.parse("Sat Mar 15 09:30:00 2025")      # datetime(2025, 3, 15, 9, 30)
parser.parse("2025-03-15T09:30:00+05:00")     # datetime(2025, 3, 15, 9, 30, tzinfo=tzoffset(None, 18000))
parser.parse("next Thursday")                 # relative to today (may be ambiguous)
```

#### Key Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `timestr` | `str` | (required) | The date/time string to parse |
| `dayfirst` | `bool` | `False` | Interpret first ambiguous number as day (EU style) |
| `yearfirst` | `bool` | `False` | Interpret first ambiguous number as year |
| `fuzzy` | `bool` | `False` | Allow fuzzy parsing, ignoring unrecognized tokens |
| `default` | `datetime` | today at midnight | Fill in missing fields from this datetime |
| `ignoretz` | `bool` | `False` | Ignore timezone information in the string |

```python
# Ambiguous date: is 01/02/03 Jan 2 or Feb 1?
parser.parse("01/02/03")                     # datetime(2003, 1, 2, 0, 0)  -- default
parser.parse("01/02/03", dayfirst=True)      # datetime(2003, 2, 1, 0, 0)  -- EU style
parser.parse("01/02/03", yearfirst=True)     # datetime(2001, 2, 3, 0, 0)  -- ISO-ish

# Fuzzy parsing: extract date from surrounding text
parser.parse("The meeting is on March 15, 2025 at 3pm", fuzzy=True)
# datetime(2025, 3, 15, 15, 0)
```

### `relativedelta` -- Date Arithmetic

`relativedelta` extends `timedelta` with support for months, years, and complex calendar arithmetic.

```python
from dateutil.relativedelta import relativedelta
from datetime import datetime

now = datetime(2025, 1, 31, 12, 0)

# Add months (handles month-end rollover correctly)
now + relativedelta(months=1)    # datetime(2025, 2, 28, 12, 0)  -- NOT March 3!
now + relativedelta(months=2)    # datetime(2025, 3, 31, 12, 0)

# Add years
now + relativedelta(years=1)     # datetime(2026, 1, 31, 12, 0)

# Combine multiple units
now + relativedelta(years=1, months=2, days=3, hours=4)
# datetime(2026, 4, 3, 16, 0)

# Absolute values (replace fields)
now + relativedelta(month=6, day=15, hour=9)
# datetime(2025, 6, 15, 9, 0)  -- set to June 15 at 9 AM

# Compute difference between dates
delta = relativedelta(datetime(2025, 6, 15), datetime(2023, 1, 10))
# relativedelta(years=+2, months=+5, days=+5)
delta.years    # 2
delta.months   # 5
delta.days     # 5
```

#### Relative vs Absolute Arguments

| Argument | Type | Effect |
|----------|------|--------|
| `years`, `months`, `weeks`, `days`, `hours`, `minutes`, `seconds` | Relative (plural) | Added to the date |
| `year`, `month`, `day`, `hour`, `minute`, `second` | Absolute (singular) | Replaces the field |
| `weekday` | Special | Advance to the given weekday |

```python
from dateutil.relativedelta import relativedelta, MO, TU, WE, TH, FR, SA, SU

dt = datetime(2025, 3, 12)  # Wednesday

# Next Monday
dt + relativedelta(weekday=MO)     # datetime(2025, 3, 17)

# Next Monday (or this Monday if today is Monday)
dt + relativedelta(weekday=MO(0))  # same as MO

# Second Friday from now
dt + relativedelta(weekday=FR(+2)) # datetime(2025, 3, 21)

# Last Monday
dt + relativedelta(weekday=MO(-1)) # datetime(2025, 3, 10)
```

### `rrule` -- Recurrence Rules

Generates recurring datetimes following RFC 2445 (iCalendar) recurrence rules.

```python
from dateutil.rrule import rrule, DAILY, WEEKLY, MONTHLY, YEARLY, MO, TU, WE, TH, FR
from datetime import datetime

start = datetime(2025, 1, 1)

# Every day for 5 days
list(rrule(DAILY, count=5, dtstart=start))
# [datetime(2025, 1, 1), ..., datetime(2025, 1, 5)]

# Every Monday and Wednesday for 4 occurrences
list(rrule(WEEKLY, count=4, byweekday=(MO, WE), dtstart=start))
# [datetime(2025, 1, 1), datetime(2025, 1, 6), datetime(2025, 1, 8), datetime(2025, 1, 13)]

# First day of every month in 2025
list(rrule(MONTHLY, count=12, bymonthday=1, dtstart=start))

# Every year on March 15
list(rrule(YEARLY, count=5, bymonth=3, bymonthday=15, dtstart=start))

# With an end date (until)
list(rrule(DAILY, dtstart=start, until=datetime(2025, 1, 10)))
```

#### `rruleset` -- Combining Rules

```python
from dateutil.rrule import rruleset, rrule, DAILY
from datetime import datetime

rset = rruleset()
rset.rrule(rrule(DAILY, count=5, dtstart=datetime(2025, 1, 1)))
rset.exdate(datetime(2025, 1, 3))  # exclude Jan 3

list(rset)
# [datetime(2025, 1, 1), datetime(2025, 1, 2), datetime(2025, 1, 4), datetime(2025, 1, 5)]
```

#### `rrulestr` -- Parse iCalendar RRULE Strings

```python
from dateutil.rrule import rrulestr

rule = rrulestr("RRULE:FREQ=WEEKLY;BYDAY=MO,WE,FR;COUNT=6", dtstart=datetime(2025, 1, 1))
list(rule)
```

### `tz` -- Timezone Utilities

```python
from dateutil import tz
from datetime import datetime

# Get local timezone
local = tz.tzlocal()

# Get timezone by name (uses system tz database)
eastern = tz.gettz("America/New_York")
utc = tz.UTC

# Create timezone-aware datetimes
dt = datetime(2025, 6, 15, 12, 0, tzinfo=eastern)

# Convert between timezones
dt_utc = dt.astimezone(utc)
print(dt_utc)  # 2025-06-15 16:00:00+00:00

# Fixed offset
plus_530 = tz.tzoffset("IST", 5.5 * 3600)
```

### `easter` -- Easter Date Calculation

```python
from dateutil.easter import easter, EASTER_WESTERN, EASTER_ORTHODOX, EASTER_JULIAN

easter(2025)                           # date(2025, 4, 20)
easter(2025, EASTER_ORTHODOX)          # date(2025, 4, 20) -- same in 2025
easter(2025, EASTER_JULIAN)            # date(2025, 4, 7)
```
