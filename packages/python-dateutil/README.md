# python-dateutil

Powerful **extensions to the standard `datetime` module** in Python.

`python-dateutil` provides flexible date parsing, relative deltas for complex date arithmetic, recurrence rules (RFC 2445 / iCalendar), timezone handling, and Easter computation. It is one of the most widely depended-upon Python packages and is a dependency of pandas, matplotlib, and many other popular libraries.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `python-dateutil` |
| **Latest version** | 2.9.0.post0 (released March 2024) |
| **Python support** | Python >= 3.3 |
| **License** | Apache 2.0 + BSD 3-Clause (dual) |
| **Repository** | https://github.com/dateutil/dateutil |
| **Documentation** | https://dateutil.readthedocs.io |

The import name is `dateutil` (not `python_dateutil`).

---

## Installation

```bash
pip install python-dateutil
```

Depends on `six` (for Python 2/3 compat, retained for legacy reasons in some versions). Modern versions (2.8.2+) work with Python 3 only.

```python
import dateutil
from dateutil import parser, relativedelta, rrule, tz, easter
```

---

## Core API Reference

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

---

## Gotchas and Common Mistakes

### 1. `parser.parse()` Guesses Ambiguous Dates

```python
# What is "01/02/03"?
parser.parse("01/02/03")                # 2003-01-02 (American: month/day/year)
parser.parse("01/02/03", dayfirst=True) # 2003-02-01 (European: day/month/year)
```

Always set `dayfirst` or `yearfirst` explicitly when parsing user input from different locales.

### 2. `relativedelta(months=1)` Handles Month-End Correctly

Adding one month to Jan 31 gives Feb 28 (or 29), NOT March 3. This is usually desired but can surprise if you expect arithmetic overflow:

```python
datetime(2025, 1, 31) + relativedelta(months=1)  # Feb 28, not Mar 3
```

### 3. Singular vs Plural Arguments in `relativedelta`

- **Plural** (`months=2`): adds 2 months (relative)
- **Singular** (`month=2`): sets the month to February (absolute)

Mixing these up is a very common bug:

```python
# WRONG: this SETS month=1, it does not ADD 1 month
datetime(2025, 6, 15) + relativedelta(month=1)  # datetime(2025, 1, 15)

# CORRECT: ADD 1 month
datetime(2025, 6, 15) + relativedelta(months=1)  # datetime(2025, 7, 15)
```

### 4. `parser.parse()` Fills Missing Fields from Today

If the input string does not contain a year, the current year is assumed. Similarly for month and day:

```python
# If today is 2025-06-15:
parser.parse("March 10")  # datetime(2025, 3, 10, 0, 0)  -- year from today

# Override with default parameter:
parser.parse("March 10", default=datetime(2020, 1, 1))  # datetime(2020, 3, 10, 0, 0)
```

### 5. Install Package is `python-dateutil`, Import is `dateutil`

```bash
pip install python-dateutil   # package name with "python-" prefix
```
```python
import dateutil               # import without "python-" prefix
```

---

## Complete Examples

### Example 1: Business Day Calculation

```python
from dateutil.relativedelta import relativedelta, MO, FR
from dateutil.rrule import rrule, DAILY, MO as rMO, TU, WE, TH, FR as rFR
from datetime import datetime

start = datetime(2025, 3, 1)

# Next 10 business days
bdays = list(rrule(
    DAILY,
    count=10,
    byweekday=(rMO, TU, WE, TH, rFR),
    dtstart=start
))

for d in bdays:
    print(d.strftime("%Y-%m-%d %A"))
```

### Example 2: Age Calculation

```python
from dateutil.relativedelta import relativedelta
from datetime import datetime

birth = datetime(1990, 7, 25)
today = datetime(2025, 3, 15)

age = relativedelta(today, birth)
print(f"Age: {age.years} years, {age.months} months, {age.days} days")
# Age: 34 years, 7 months, 18 days
```

### Example 3: Parse Multiple Date Formats

```python
from dateutil import parser

samples = [
    "2025-03-15",
    "March 15, 2025",
    "15-Mar-2025",
    "3/15/25",
    "Sat, 15 Mar 2025 14:30:00 +0000",
    "20250315T143000Z",
]

for s in samples:
    dt = parser.parse(s)
    print(f"{s:45s} -> {dt.isoformat()}")
```

### Example 4: Monthly Report Dates (Last Friday of Each Month)

```python
from dateutil.rrule import rrule, MONTHLY, FR
from datetime import datetime

# Last Friday of every month in 2025
dates = list(rrule(
    MONTHLY,
    count=12,
    byweekday=FR(-1),
    dtstart=datetime(2025, 1, 1)
))

for d in dates:
    print(d.strftime("%Y-%m-%d %A"))
# 2025-01-31 Friday
# 2025-02-28 Friday
# 2025-03-28 Friday
# ...
```

### Example 5: Working with Timezones

```python
from dateutil import tz
from datetime import datetime

# Schedule a meeting at 2 PM Eastern, find the time in other zones
eastern = tz.gettz("America/New_York")
pacific = tz.gettz("America/Los_Angeles")
london = tz.gettz("Europe/London")
tokyo = tz.gettz("Asia/Tokyo")

meeting = datetime(2025, 6, 15, 14, 0, tzinfo=eastern)

for name, zone in [("New York", eastern), ("Los Angeles", pacific),
                    ("London", london), ("Tokyo", tokyo)]:
    local_time = meeting.astimezone(zone)
    print(f"{name:15s}: {local_time.strftime('%Y-%m-%d %I:%M %p %Z')}")
```

---

## See Also

- [dateutil on GitHub](https://github.com/dateutil/dateutil)
- [Full Documentation](https://dateutil.readthedocs.io)
- [RFC 2445 (iCalendar)](https://www.rfc-editor.org/rfc/rfc2445) -- the standard behind `rrule`
