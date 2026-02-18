# python-dateutil — Examples & Gotchas

> Part of the python-dateutil skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. `parser.parse()` Guesses Ambiguous Dates](#1-parserparse-guesses-ambiguous-dates)
  - [2. `relativedelta(months=1)` Handles Month-End Correctly](#2-relativedeltamonths1-handles-month-end-correctly)
  - [3. Singular vs Plural Arguments in `relativedelta`](#3-singular-vs-plural-arguments-in-relativedelta)
  - [4. `parser.parse()` Fills Missing Fields from Today](#4-parserparse-fills-missing-fields-from-today)
  - [5. Install Package is `python-dateutil`, Import is `dateutil`](#5-install-package-is-python-dateutil-import-is-dateutil)
- [Complete Examples](#complete-examples)
  - [Example 1: Business Day Calculation](#example-1-business-day-calculation)
  - [Example 2: Age Calculation](#example-2-age-calculation)
  - [Example 3: Parse Multiple Date Formats](#example-3-parse-multiple-date-formats)
  - [Example 4: Monthly Report Dates (Last Friday of Each Month)](#example-4-monthly-report-dates-last-friday-of-each-month)
  - [Example 5: Working with Timezones](#example-5-working-with-timezones)
- [See Also](#see-also)

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
