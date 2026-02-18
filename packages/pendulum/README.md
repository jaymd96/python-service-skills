# pendulum

## Overview

Pendulum is a Python package that provides a drop-in replacement for the standard library `datetime` module. It offers timezone-aware datetime objects by default, an immutable API where all mutation operations return new instances, and a fluent interface for date/time arithmetic, formatting, and comparison.

Key design principles:

- **Always timezone-aware** -- every `DateTime` instance carries timezone information; there is no concept of a "naive" pendulum datetime.
- **Immutable** -- methods like `.add()` and `.subtract()` return new objects rather than mutating in place.
- **Fluent API** -- operations can be chained naturally: `pendulum.now().subtract(days=3).start_of('day')`.
- **Drop-in compatible** -- `pendulum.DateTime` is a subclass of `datetime.datetime`, so it works anywhere a stdlib datetime is expected.
- **Human-readable** -- built-in support for relative time strings like "2 hours ago" with locale awareness.

**Repository:** https://github.com/sdispater/pendulum
**Documentation:** https://pendulum.eustace.io
**License:** MIT

---

## Latest Stable Version

Pendulum **3.0.0** was released as the latest major version. The 3.x line is the current stable series.

- Pendulum 3.x requires **Python 3.8+**.
- Pendulum 2.x (last release: 2.1.2) is the legacy line for older Python versions.

Check your installed version:

```python
import pendulum
print(pendulum.__version__)
```

---

## Installation

```bash
pip install pendulum
```

Pendulum 3.x ships with pre-compiled Rust extensions for performance. If the binary wheel is not available for your platform, it falls back to a pure-Python implementation.

To install a specific version:

```bash
pip install pendulum==3.0.0
pip install "pendulum>=3,<4"    # any 3.x release
pip install "pendulum>=2,<3"    # legacy 2.x
```

---

## Core API

### `pendulum.now(tz=None)`

Returns the current date and time as a timezone-aware `DateTime`. If `tz` is omitted, the local system timezone is used.

```python
import pendulum

now = pendulum.now()                      # local timezone
now_utc = pendulum.now("UTC")             # UTC
now_paris = pendulum.now("Europe/Paris")  # specific timezone

print(now)           # e.g. 2026-02-17T14:30:00.123456-05:00
print(now.timezone_name)  # e.g. "America/New_York"
```

### `pendulum.today()`, `pendulum.tomorrow()`, `pendulum.yesterday()`

Convenience functions that return `DateTime` objects set to midnight (00:00:00) of the respective day.

```python
today = pendulum.today()                  # today at 00:00:00 in local tz
today_utc = pendulum.today("UTC")         # today at 00:00:00 UTC
tomorrow = pendulum.tomorrow("US/Eastern")
yesterday = pendulum.yesterday()
```

### `pendulum.datetime(year, month, day, hour=0, minute=0, second=0, microsecond=0, tz=None)`

Creates a `DateTime` instance from explicit components. If `tz` is `None`, UTC is used.

```python
dt = pendulum.datetime(2026, 3, 15)
# DateTime(2026, 3, 15, 0, 0, 0, tzinfo=Timezone('UTC'))

dt = pendulum.datetime(2026, 3, 15, 14, 30, tz="America/Chicago")
# DateTime(2026, 3, 15, 14, 30, 0, tzinfo=Timezone('America/Chicago'))
```

### `pendulum.parse(text, **options)`

Parses a date/time string into a `DateTime`. Supports ISO 8601 and many common formats.

```python
dt = pendulum.parse("2026-03-15")
# DateTime(2026, 3, 15, 0, 0, 0, tzinfo=Timezone('UTC'))

dt = pendulum.parse("2026-03-15T14:30:00+02:00")
# DateTime(2026, 3, 15, 14, 30, 0, tzinfo=...)

dt = pendulum.parse("2026-03-15", tz="America/New_York")
# DateTime(2026, 3, 15, 0, 0, 0, tzinfo=Timezone('America/New_York'))
```

By default, `parse()` operates in **strict mode** and expects well-formed strings. To enable looser parsing:

```python
dt = pendulum.parse("March 15, 2026", strict=False)
```

**Note:** In pendulum 3.x, `strict=False` parsing capabilities may differ from 2.x. For maximum control, prefer `from_format()`.

### `pendulum.from_format(string, fmt, tz=None)`

Parses a date/time string using an explicit format. Format tokens follow pendulum's own token system (similar to but not identical to `strftime`).

```python
dt = pendulum.from_format("15/03/2026 14:30", "DD/MM/YYYY HH:mm")
dt = pendulum.from_format("2026-03-15", "YYYY-MM-DD", tz="Europe/London")
```

### `pendulum.from_timestamp(timestamp, tz=None)`

Creates a `DateTime` from a Unix timestamp.

```python
dt = pendulum.from_timestamp(1773849600)
# DateTime in UTC

dt = pendulum.from_timestamp(1773849600, tz="Asia/Tokyo")
# DateTime converted to Asia/Tokyo
```

### `pendulum.duration(...)`

Creates a `Duration` object representing an absolute span of time. This is pendulum's replacement for `datetime.timedelta`.

```python
dur = pendulum.duration(days=3, hours=5, minutes=30)
print(dur.in_hours())  # total hours as a float/int

dur = pendulum.duration(years=1, months=6)
# Unlike timedelta, pendulum durations understand months and years
```

### `pendulum.interval(...)`

`interval()` is an alias for `duration()` in pendulum 2.x. In pendulum 3.x, the preferred name is `duration()`.

```python
# pendulum 2.x
iv = pendulum.interval(weeks=2, days=3)

# pendulum 3.x -- use duration() instead
dur = pendulum.duration(weeks=2, days=3)
```

### `pendulum.period(start, end)`

Creates a `Period` representing the span between two `DateTime` instances. Unlike a `Duration`, a `Period` retains the start and end context and can be iterated over.

```python
start = pendulum.datetime(2026, 1, 1)
end = pendulum.datetime(2026, 3, 31)
period = pendulum.period(start, end)

print(period.in_days())     # total days between start and end
print(period.in_months())   # total months

# Iterate over the period
for dt in period.range("days"):
    print(dt)  # prints each day from Jan 1 to Mar 31

for dt in period.range("months"):
    print(dt)  # prints the 1st of each month in the range
```

---

## DateTime Operations

### `.add()` and `.subtract()` -- Fluent Arithmetic

Both methods return a **new** `DateTime` instance (the original is unchanged).

```python
dt = pendulum.datetime(2026, 1, 15, 12, 0, 0)

# Add time
future = dt.add(years=1, months=2, days=3, hours=4)
# DateTime(2027, 3, 18, 16, 0, 0)

# Subtract time
past = dt.subtract(weeks=2, hours=6)
# DateTime(2026, 1, 1, 6, 0, 0)

# Chaining
result = dt.add(days=30).subtract(hours=12).add(minutes=15)
```

Supported keyword arguments: `years`, `months`, `weeks`, `days`, `hours`, `minutes`, `seconds`, `microseconds`.

### `.diff()` -- Difference as Period

Returns a `Period` (or `Duration` depending on version) representing the difference between two datetimes.

```python
dt1 = pendulum.datetime(2026, 1, 1)
dt2 = pendulum.datetime(2026, 6, 15, 14, 30)

diff = dt1.diff(dt2)
print(diff.in_days())       # 165
print(diff.in_hours())      # 3966
print(diff.in_months())     # 5
print(diff.in_words())      # "5 months 2 weeks 14 hours 30 minutes"
```

When called without arguments, `.diff()` computes the difference from now:

```python
dt = pendulum.datetime(2025, 1, 1)
diff = dt.diff()  # difference between dt and now
```

### `.diff_for_humans()` -- Human-Readable Diffs

Returns a natural-language string describing the difference relative to now (or another datetime).

```python
past = pendulum.now().subtract(hours=3)
print(past.diff_for_humans())  # "3 hours ago"

future = pendulum.now().add(days=2)
print(future.diff_for_humans())  # "in 2 days"

# Relative to another datetime
dt1 = pendulum.datetime(2026, 1, 1)
dt2 = pendulum.datetime(2026, 3, 15)
print(dt1.diff_for_humans(dt2))  # "2 months before"
```

Supports locales:

```python
pendulum.set_locale("fr")
print(past.diff_for_humans())  # "il y a 3 heures"

# Or per-call:
print(past.diff_for_humans(locale="de"))  # "vor 3 Stunden"
```

### `.format()` -- Custom Formatting

Uses pendulum's own format tokens (not `strftime` by default).

```python
dt = pendulum.datetime(2026, 3, 15, 14, 30, 45)

dt.format("YYYY-MM-DD HH:mm:ss")       # "2026-03-15 14:30:45"
dt.format("dddd, MMMM D, YYYY")        # "Sunday, March 15, 2026"
dt.format("DD/MM/YYYY")                 # "15/03/2026"
dt.format("hh:mm A")                    # "02:30 PM"
dt.format("Do [of] MMMM, YYYY")        # "15th of March, 2026"
```

You can also use Python's `strftime`:

```python
dt.strftime("%Y-%m-%d %H:%M:%S")       # "2026-03-15 14:30:45"
```

### Preset Format Methods

```python
dt = pendulum.datetime(2026, 3, 15, 14, 30, 45, tz="UTC")

dt.to_iso8601_string()          # "2026-03-15T14:30:45+00:00"
dt.to_datetime_string()         # "2026-03-15 14:30:45"
dt.to_date_string()             # "2026-03-15"
dt.to_time_string()             # "14:30:45"
dt.to_day_datetime_string()     # "Sun, Mar 15, 2026 2:30 PM"
dt.to_formatted_date_string()   # "Mar 15, 2026"
dt.to_atom_string()             # "2026-03-15T14:30:45+00:00"
dt.to_cookie_string()           # "Sunday, 15-Mar-2026 14:30:45 UTC"
dt.to_rss_string()              # "Sun, 15 Mar 2026 14:30:45 +0000"
dt.to_w3c_string()              # "2026-03-15T14:30:45+00:00"
```

### `.start_of()` and `.end_of()` -- Boundary Operations

Returns a new `DateTime` set to the start or end of the given unit.

```python
dt = pendulum.datetime(2026, 3, 15, 14, 30, 45)

dt.start_of("year")    # 2026-01-01 00:00:00
dt.start_of("month")   # 2026-03-01 00:00:00
dt.start_of("week")    # 2026-03-09 00:00:00 (Monday)
dt.start_of("day")     # 2026-03-15 00:00:00
dt.start_of("hour")    # 2026-03-15 14:00:00

dt.end_of("year")      # 2026-12-31 23:59:59.999999
dt.end_of("month")     # 2026-03-31 23:59:59.999999
dt.end_of("week")      # 2026-03-15 23:59:59.999999 (Sunday)
dt.end_of("day")       # 2026-03-15 23:59:59.999999
```

**Note:** Pendulum considers Monday as the first day of the week by default.

### `.is_past()`, `.is_future()`, `.is_today()` and Other Comparisons

```python
dt = pendulum.datetime(2020, 6, 15)
dt.is_past()          # True (assuming current date is 2026)
dt.is_future()        # False
dt.is_today()         # False

now = pendulum.now()
now.is_today()        # True

# Day-of-week checks
dt.is_monday()        # True/False depending on the date
dt.is_weekend()       # True if Saturday or Sunday

# Other comparisons
dt.is_leap_year()     # True/False
dt.is_same_day(other_dt)
dt.is_birthday(other_dt)  # same month and day
```

### `.in_timezone()` / `.in_tz()` -- Timezone Conversion

Converts the datetime to a different timezone. The underlying instant remains the same; only the representation changes.

```python
utc = pendulum.now("UTC")
eastern = utc.in_timezone("America/New_York")
tokyo = utc.in_tz("Asia/Tokyo")  # .in_tz() is a shorthand alias

print(utc)      # 2026-02-17T19:30:00+00:00
print(eastern)  # 2026-02-17T14:30:00-05:00
print(tokyo)    # 2026-02-18T04:30:00+09:00

# All three represent the same instant:
assert utc == eastern == tokyo
```

### `.set()` -- Replacing Components

Returns a new `DateTime` with specified components replaced.

```python
dt = pendulum.datetime(2026, 3, 15, 14, 30, 45)

dt.set(hour=0, minute=0, second=0)   # 2026-03-15 00:00:00
dt.set(year=2030)                     # 2030-03-15 14:30:45
dt.set(month=12, day=25)             # 2026-12-25 14:30:45
```

You can also use `.replace()` (inherited from `datetime.datetime`) but `.set()` is the pendulum-idiomatic approach.

---

## Timezone Handling

### `pendulum.timezone(name)`

Creates a `Timezone` object.

```python
tz = pendulum.timezone("America/New_York")
print(tz)  # Timezone('America/New_York')
```

### `Timezone` Class API

```python
tz = pendulum.timezone("Europe/London")

# Convert a datetime to this timezone
dt = pendulum.now("UTC")
localized = tz.convert(dt)

# Get the timezone name
print(tz.name)  # "Europe/London"
```

### DST Transition Handling

Pendulum handles Daylight Saving Time transitions correctly. When adding or subtracting time across a DST boundary, the wall clock time adjusts properly.

```python
# Spring forward: clocks jump from 2:00 AM to 3:00 AM
dt = pendulum.datetime(2026, 3, 8, 1, 30, tz="America/New_York")
after = dt.add(hours=1)
print(after)  # 2026-03-08T03:30:00-04:00 (EDT, skipped 2:00-3:00)

# Fall back: clocks repeat 1:00 AM - 2:00 AM
dt = pendulum.datetime(2026, 11, 1, 0, 30, tz="America/New_York")
after = dt.add(hours=1)
print(after)  # 2026-11-01T01:30:00-04:00 (still EDT)
after2 = dt.add(hours=2)
print(after2) # 2026-11-01T01:30:00-05:00 (now EST)
```

Pendulum resolves ambiguous times by defaulting to the **post-transition** (later) offset. You can control this:

```python
# During fall-back ambiguity:
dt = pendulum.datetime(2026, 11, 1, 1, 30, tz="America/New_York")
# By default this is the second occurrence (EST, -05:00)
```

### Available Timezone Names

Pendulum uses the IANA timezone database. All standard timezone names are supported:

```python
# Examples of valid timezone names:
"UTC"
"America/New_York"
"America/Chicago"
"America/Los_Angeles"
"Europe/London"
"Europe/Paris"
"Europe/Berlin"
"Asia/Tokyo"
"Asia/Shanghai"
"Australia/Sydney"
"Pacific/Auckland"
"US/Eastern"         # alias for America/New_York
"US/Central"         # alias for America/Chicago
"US/Pacific"         # alias for America/Los_Angeles
```

You can also use fixed offsets:

```python
dt = pendulum.now("+05:30")    # UTC+05:30
dt = pendulum.now("-08:00")    # UTC-08:00
```

---

## Duration and Period

### Duration -- Absolute Time Spans

A `Duration` represents a length of time without reference to any specific starting or ending point. It extends `datetime.timedelta` and adds support for months and years.

```python
dur = pendulum.duration(
    years=1,
    months=6,
    weeks=2,
    days=3,
    hours=12,
    minutes=30,
    seconds=15,
    microseconds=500000
)

# Accessor properties
print(dur.years)         # 1
print(dur.months)        # 6
print(dur.weeks)         # 2
print(dur.days)          # 3 (remaining days, not total)
print(dur.hours)         # 12
print(dur.minutes)       # 30
print(dur.seconds)       # 15
print(dur.microseconds)  # 500000

# Total conversions
print(dur.in_years())        # approximate total years
print(dur.in_months())       # approximate total months
print(dur.in_weeks())        # total weeks
print(dur.in_days())         # total days
print(dur.in_hours())        # total hours
print(dur.in_minutes())      # total minutes
print(dur.in_seconds())      # total seconds

# Human-readable
print(dur.in_words())    # "1 year 6 months 2 weeks 3 days 12 hours 30 minutes 15 seconds"
```

**Important:** Because months and years have variable lengths, `in_days()` and similar conversions for durations that include months/years are approximate. The exact number of days depends on which specific months and years are involved -- this is only resolved when a `Duration` is applied to a specific `DateTime` via `.add()` or `.subtract()`.

### Period -- Time Between Two DateTimes

A `Period` is the difference between two `DateTime` instances. It is context-aware, meaning it knows the exact calendar dates involved, so conversions are precise.

```python
start = pendulum.datetime(2026, 1, 15)
end = pendulum.datetime(2026, 9, 20, 18, 45)

period = end - start
# Or explicitly:
period = pendulum.period(start, end)

print(period.in_years())     # 0
print(period.in_months())    # 8
print(period.in_weeks())     # 35
print(period.in_days())      # 248
print(period.in_hours())     # 5970
print(period.in_minutes())   # 358245
print(period.in_seconds())   # 21494700

print(period.in_words())
# "8 months 5 days 18 hours 45 minutes"
```

### Iterating Over Periods

Use `.range()` to iterate over a period at a given step granularity.

```python
start = pendulum.datetime(2026, 1, 1)
end = pendulum.datetime(2026, 6, 30)
period = pendulum.period(start, end)

# Daily iteration
for dt in period.range("days"):
    print(dt.to_date_string())  # "2026-01-01", "2026-01-02", ...

# Weekly iteration
for dt in period.range("weeks"):
    print(dt.to_date_string())  # "2026-01-01", "2026-01-08", ...

# Monthly iteration
for dt in period.range("months"):
    print(dt.to_date_string())  # "2026-01-01", "2026-02-01", ...

# Custom step
for dt in period.range("days", 3):  # every 3 days
    print(dt.to_date_string())
```

Valid range units: `"years"`, `"months"`, `"weeks"`, `"days"`, `"hours"`, `"minutes"`, `"seconds"`.

---

## Formatting and Parsing

### Format Tokens Reference

Pendulum uses its own token system in `.format()` and `from_format()`:

| Token  | Output                          | Example              |
|--------|---------------------------------|----------------------|
| `YYYY` | 4-digit year                    | `2026`               |
| `YY`   | 2-digit year                    | `26`                 |
| `MMMM` | Full month name                 | `March`              |
| `MMM`  | Abbreviated month name          | `Mar`                |
| `MM`   | Zero-padded month               | `03`                 |
| `M`    | Month (no padding)              | `3`                  |
| `DD`   | Zero-padded day of month        | `05`                 |
| `D`    | Day of month (no padding)       | `5`                  |
| `Do`   | Day with ordinal                | `5th`                |
| `dddd` | Full day name                   | `Sunday`             |
| `ddd`  | Abbreviated day name            | `Sun`                |
| `dd`   | 2-letter day name               | `Su`                 |
| `d`    | Day of week (0=Sunday)          | `0`                  |
| `HH`   | 24-hour zero-padded             | `14`                 |
| `H`    | 24-hour (no padding)            | `14`                 |
| `hh`   | 12-hour zero-padded             | `02`                 |
| `h`    | 12-hour (no padding)            | `2`                  |
| `mm`   | Zero-padded minutes             | `05`                 |
| `m`    | Minutes (no padding)            | `5`                  |
| `ss`   | Zero-padded seconds             | `09`                 |
| `s`    | Seconds (no padding)            | `9`                  |
| `S`    | Tenths of a second              | `1`                  |
| `SS`   | Hundredths of a second          | `12`                 |
| `SSS`  | Milliseconds                    | `123`                |
| `SSSS..`| Micro/nanoseconds (up to 9)   | `123456`             |
| `A`    | AM/PM                           | `PM`                 |
| `Z`    | UTC offset                      | `+02:00`             |
| `ZZ`   | UTC offset (no colon)           | `+0200`              |
| `z`    | Timezone abbreviation           | `EST`                |
| `X`    | Unix timestamp                  | `1773849600`         |
| `x`    | Unix timestamp in ms            | `1773849600000`      |

Use square brackets to escape literal text in format strings:

```python
dt.format("[Today is] dddd")  # "Today is Sunday"
```

### Locale Support

Pendulum supports many locales for formatting and `diff_for_humans()`.

```python
import pendulum

# Global locale
pendulum.set_locale("fr")
dt = pendulum.now()
print(dt.format("dddd D MMMM YYYY"))  # "mardi 17 fevrier 2026"
print(dt.diff_for_humans())            # "il y a quelques secondes"

# Reset
pendulum.set_locale("en")

# Per-call locale (diff_for_humans)
print(dt.diff_for_humans(locale="es"))  # "hace unos segundos"
```

Supported locales include: `en`, `fr`, `de`, `es`, `it`, `pt`, `pt_br`, `nl`, `da`, `nb`, `sv`, `fi`, `pl`, `ru`, `uk`, `zh`, `ja`, `ko`, `ar`, `tr`, and many more.

### Strict vs. Loose Parsing

```python
# Strict parsing (default) -- expects ISO 8601 or well-known formats
pendulum.parse("2026-03-15T14:30:00")       # works
pendulum.parse("2026-03-15")                 # works
pendulum.parse("March 15, 2026")             # raises ValueError in strict mode

# Loose parsing -- attempts to parse more formats
pendulum.parse("March 15, 2026", strict=False)   # works
pendulum.parse("next tuesday", strict=False)       # may work (depends on version)

# Explicit format -- most reliable approach
pendulum.from_format("15-Mar-2026", "DD-MMM-YYYY")
```

---

## Drop-in Compatibility

`pendulum.DateTime` extends `datetime.datetime`, so it can be used as a drop-in replacement in most contexts.

```python
import pendulum
from datetime import datetime

dt = pendulum.now()

# isinstance checks pass
assert isinstance(dt, datetime)  # True

# Works with standard library functions
import calendar
calendar.isleap(dt.year)

# Works with strftime
dt.strftime("%Y-%m-%d %H:%M:%S")

# Works with timedelta arithmetic
from datetime import timedelta
result = dt + timedelta(days=5)
# Note: result is a stdlib datetime, NOT a pendulum DateTime
# Use dt.add(days=5) to stay in pendulum's world

# Converting stdlib datetime to pendulum
stdlib_dt = datetime(2026, 3, 15, 14, 30, 0)
pdt = pendulum.instance(stdlib_dt, tz="UTC")
```

**Important:** Operations that mix pendulum `DateTime` with stdlib `timedelta` may return a plain `datetime.datetime`. Always use pendulum's `.add()` / `.subtract()` to guarantee a `pendulum.DateTime` return type.

---

## Gotchas and Common Mistakes

### Pendulum 2.x vs 3.x Breaking Changes

The transition from pendulum 2.x to 3.x introduced several breaking changes:

| Topic | 2.x | 3.x |
|-------|-----|-----|
| `pendulum.interval()` | Creates a Duration | Removed or renamed; use `pendulum.duration()` |
| `pendulum.period()` factory | Available | Behavior may differ; subtraction of two DateTimes returns a Duration in some cases |
| `Duration.in_words()` | Available | Check availability; API may have changed |
| Parsing behavior | Uses `dateutil` internally | Uses custom Rust-based parser |
| `DateTime.timezone` property | Returns `Timezone` object | Returns `Timezone` object (property name unchanged, but internal representation changed) |
| `pendulum.set_test_now()` | Available for mocking | Still available but verify API |
| `DateTime.int_timestamp` | Property | May be renamed or removed; use `int(dt.timestamp())` |

When migrating from 2.x to 3.x, test all date/time operations thoroughly. Pin your version in `requirements.txt` or `pyproject.toml` to avoid unexpected upgrades.

### Serialization / JSON

Pendulum `DateTime` objects are **not** JSON-serializable by default:

```python
import json
import pendulum

dt = pendulum.now()

# This raises TypeError:
# json.dumps({"created_at": dt})

# Solutions:

# 1. Convert to ISO string
json.dumps({"created_at": dt.to_iso8601_string()})

# 2. Custom encoder
class PendulumEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, pendulum.DateTime):
            return obj.to_iso8601_string()
        if isinstance(obj, pendulum.Duration):
            return obj.total_seconds()
        return super().default(obj)

json.dumps({"created_at": dt}, cls=PendulumEncoder)

# 3. Using str()
json.dumps({"created_at": str(dt)})
```

If you use **orjson**, it handles `datetime` subclasses natively:

```python
import orjson
orjson.dumps({"created_at": dt})  # works because DateTime is a datetime subclass
```

### Interaction with SQLAlchemy

SQLAlchemy's `DateTime` column type works with pendulum, but returned objects are stdlib `datetime` instances (from the database driver), not pendulum `DateTime` objects. You need to convert explicitly:

```python
import pendulum
from sqlalchemy import Column, DateTime, Integer, create_engine
from sqlalchemy.orm import declarative_base, Session

Base = declarative_base()

class Event(Base):
    __tablename__ = "events"
    id = Column(Integer, primary_key=True)
    created_at = Column(DateTime(timezone=True))

# When saving, pendulum DateTime works directly:
event = Event(created_at=pendulum.now("UTC"))

# When reading, you get a stdlib datetime; convert it:
event = session.query(Event).first()
created = pendulum.instance(event.created_at, tz="UTC")
```

For automatic conversion, consider a custom type decorator:

```python
from sqlalchemy import TypeDecorator, DateTime

class PendulumDateTime(TypeDecorator):
    impl = DateTime(timezone=True)
    cache_ok = True

    def process_result_value(self, value, dialect):
        if value is not None:
            return pendulum.instance(value, tz="UTC")
        return value
```

### Interaction with attrs / cattrs

When using `attrs` or `cattrs`, you may need custom structuring/unstructuring hooks:

```python
import attrs
import cattrs
import pendulum

@attrs.define
class Event:
    name: str
    timestamp: pendulum.DateTime

# cattrs does not know how to structure pendulum.DateTime by default
converter = cattrs.Converter()

# Register hooks
converter.register_structure_hook(
    pendulum.DateTime,
    lambda v, _: pendulum.parse(v) if isinstance(v, str) else pendulum.instance(v)
)
converter.register_unstructure_hook(
    pendulum.DateTime,
    lambda v: v.to_iso8601_string()
)

# Now serialization works
event = Event(name="Launch", timestamp=pendulum.now())
data = converter.unstructure(event)
# {"name": "Launch", "timestamp": "2026-02-17T14:30:00+00:00"}

restored = converter.structure(data, Event)
```

### Immutability -- add/subtract Return New Objects

This is the single most common mistake for newcomers:

```python
dt = pendulum.now()

# WRONG: discards the return value
dt.add(days=5)
print(dt)  # still the original time!

# CORRECT: capture the returned value
new_dt = dt.add(days=5)
print(new_dt)  # 5 days later

# The original is unchanged
assert dt != new_dt
```

### Performance vs. stdlib datetime

Pendulum is slower than stdlib `datetime` for basic operations due to timezone handling, immutability, and the richer API. Rough performance characteristics:

- **Object creation:** ~3-10x slower than `datetime.datetime()` due to timezone resolution.
- **Arithmetic:** ~2-5x slower due to immutable copy-on-write and timezone-aware math.
- **Parsing:** Comparable or faster for ISO 8601 (Rust parser in 3.x); slower for free-form parsing.
- **Formatting:** Slightly slower due to the custom token system.

For hot loops processing millions of datetime objects, consider using stdlib `datetime` and converting to pendulum only at API boundaries. For most applications, the performance difference is negligible.

### Pickle Support

Pendulum `DateTime` objects support pickling:

```python
import pickle
import pendulum

dt = pendulum.now("America/New_York")
data = pickle.dumps(dt)
restored = pickle.loads(data)

assert dt == restored
assert restored.timezone_name == "America/New_York"
```

This also means pendulum datetimes work with `multiprocessing`, `copy.deepcopy()`, and caching libraries that rely on pickle.

### Testing and Time Mocking

Pendulum provides a built-in mechanism for freezing time in tests:

```python
import pendulum

# Freeze time at a specific point
known = pendulum.datetime(2026, 6, 15, 12, 0, 0, tz="UTC")
pendulum.set_test_now(known)

print(pendulum.now())  # always returns 2026-06-15T12:00:00+00:00

# Clear the mock
pendulum.set_test_now()
print(pendulum.now())  # returns actual current time
```

Use this in test fixtures:

```python
import pytest
import pendulum

@pytest.fixture
def frozen_time():
    frozen = pendulum.datetime(2026, 1, 1, 0, 0, 0, tz="UTC")
    pendulum.set_test_now(frozen)
    yield frozen
    pendulum.set_test_now()

def test_something(frozen_time):
    assert pendulum.now() == frozen_time
```

---

## Complete Code Examples

### Basic DateTime Creation and Manipulation

```python
import pendulum

# Current time
now = pendulum.now()
print(f"Now: {now}")
print(f"Timezone: {now.timezone_name}")
print(f"Year: {now.year}, Month: {now.month}, Day: {now.day}")
print(f"Hour: {now.hour}, Minute: {now.minute}, Second: {now.second}")

# Specific datetime
launch = pendulum.datetime(2026, 7, 4, 10, 0, 0, tz="America/New_York")
print(f"Launch: {launch}")

# Day helpers
print(f"Today: {pendulum.today()}")
print(f"Tomorrow: {pendulum.tomorrow()}")
print(f"Yesterday: {pendulum.yesterday()}")

# Day-of-week and day-of-year
print(f"Day of week: {now.day_of_week}")        # 0=Monday ... 6=Sunday (pendulum convention)
print(f"Day of year: {now.day_of_year}")
print(f"Week of year: {now.week_of_year}")
print(f"Days in month: {now.days_in_month}")
print(f"Is leap year: {now.is_leap_year()}")
```

### Timezone Conversions

```python
import pendulum

# Create a meeting time in New York
meeting_ny = pendulum.datetime(2026, 4, 15, 9, 0, 0, tz="America/New_York")

# Convert to other timezones for remote participants
meeting_london = meeting_ny.in_timezone("Europe/London")
meeting_tokyo = meeting_ny.in_timezone("Asia/Tokyo")
meeting_sydney = meeting_ny.in_timezone("Australia/Sydney")

print(f"New York:  {meeting_ny.format('hh:mm A z (Z)')}")
print(f"London:    {meeting_london.format('hh:mm A z (Z)')}")
print(f"Tokyo:     {meeting_tokyo.format('hh:mm A z (Z)')}")
print(f"Sydney:    {meeting_sydney.format('hh:mm A z (Z)')}")

# Output:
# New York:  09:00 AM EDT (-04:00)
# London:    02:00 PM BST (+01:00)
# Tokyo:     10:00 PM JST (+09:00)
# Sydney:    11:00 PM AEST (+10:00)
```

### Date Arithmetic

```python
import pendulum

today = pendulum.today("UTC")

# Add and subtract
next_quarter = today.add(months=3)
last_year = today.subtract(years=1)
two_weeks_out = today.add(weeks=2)

print(f"Today:          {today.to_date_string()}")
print(f"Next quarter:   {next_quarter.to_date_string()}")
print(f"Last year:      {last_year.to_date_string()}")
print(f"Two weeks out:  {two_weeks_out.to_date_string()}")

# Boundary operations
print(f"Start of month: {today.start_of('month').to_date_string()}")
print(f"End of month:   {today.end_of('month').to_date_string()}")
print(f"Start of week:  {today.start_of('week').to_date_string()}")
print(f"End of quarter: {today.end_of('quarter').to_date_string()}")

# Chained operations: get the last day of next month
last_day_next_month = today.add(months=1).end_of("month")
print(f"Last day of next month: {last_day_next_month.to_date_string()}")
```

### Human-Readable Diffs

```python
import pendulum

now = pendulum.now()

# Past events
created = now.subtract(hours=2, minutes=15)
print(created.diff_for_humans())  # "2 hours ago"

updated = now.subtract(days=3)
print(updated.diff_for_humans())  # "3 days ago"

ancient = now.subtract(years=5, months=3)
print(ancient.diff_for_humans())  # "5 years ago"

# Future events
deadline = now.add(days=7)
print(deadline.diff_for_humans())  # "in 6 days"

launch = now.add(months=2, days=10)
print(launch.diff_for_humans())  # "in 2 months"

# Precise differences
start = pendulum.datetime(2026, 1, 1)
end = pendulum.datetime(2026, 12, 31, 23, 59, 59)
diff = end.diff(start)
print(f"Days: {diff.in_days()}")
print(f"Hours: {diff.in_hours()}")
print(f"Minutes: {diff.in_minutes()}")
```

### Iterating Over Date Ranges

```python
import pendulum

# Generate all Mondays in Q1 2026
q1_start = pendulum.datetime(2026, 1, 1)
q1_end = pendulum.datetime(2026, 3, 31)
period = pendulum.period(q1_start, q1_end)

mondays = [dt for dt in period.range("days") if dt.day_of_week == pendulum.MONDAY]
for monday in mondays:
    print(monday.to_date_string())

# Generate monthly report dates
for month_start in period.range("months"):
    month_end = month_start.end_of("month")
    print(f"Report period: {month_start.to_date_string()} to {month_end.to_date_string()}")

# Business days in a period (exclude weekends)
business_days = [
    dt for dt in period.range("days")
    if dt.day_of_week not in (pendulum.SATURDAY, pendulum.SUNDAY)
]
print(f"Business days in Q1 2026: {len(business_days)}")

# Hourly iteration for a single day
day_start = pendulum.datetime(2026, 3, 15, 0, 0, 0)
day_end = pendulum.datetime(2026, 3, 15, 23, 0, 0)
for hour in pendulum.period(day_start, day_end).range("hours"):
    print(hour.format("HH:mm"))
```

### Formatting and Parsing

```python
import pendulum

# Parsing various formats
dt1 = pendulum.parse("2026-03-15")
dt2 = pendulum.parse("2026-03-15T14:30:00+02:00")
dt3 = pendulum.from_format("15/03/2026", "DD/MM/YYYY")
dt4 = pendulum.from_format("Mar 15, 2026 2:30 PM", "MMM D, YYYY h:mm A")
dt5 = pendulum.from_timestamp(1773849600)

# Formatting
dt = pendulum.datetime(2026, 3, 15, 14, 30, 45, tz="America/New_York")

print(dt.format("YYYY-MM-DD"))                    # "2026-03-15"
print(dt.format("dddd, MMMM Do YYYY"))            # "Sunday, March 15th 2026"
print(dt.format("hh:mm:ss A"))                     # "02:30:45 PM"
print(dt.format("YYYY-MM-DDTHH:mm:ssZ"))           # "2026-03-15T14:30:45-04:00"
print(dt.format("[Quarter] Q, YYYY"))               # "Quarter 1, 2026"

# Preset formats
print(dt.to_iso8601_string())        # ISO 8601
print(dt.to_datetime_string())       # "2026-03-15 14:30:45"
print(dt.to_date_string())           # "2026-03-15"
print(dt.to_time_string())           # "14:30:45"
print(dt.to_formatted_date_string()) # "Mar 15, 2026"

# Locale-aware formatting
pendulum.set_locale("de")
print(dt.format("dddd, D. MMMM YYYY"))  # "Sonntag, 15. Marz 2026"
pendulum.set_locale("en")  # reset
```

### Integration with SQLAlchemy Columns

```python
import pendulum
from sqlalchemy import create_engine, Column, Integer, String, DateTime, func
from sqlalchemy.orm import declarative_base, Session

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    created_at = Column(DateTime(timezone=True))
    updated_at = Column(DateTime(timezone=True))


# Setup
engine = create_engine("sqlite:///example.db")
Base.metadata.create_all(engine)

# Creating records with pendulum datetimes
with Session(engine) as session:
    user = User(
        name="Alice",
        created_at=pendulum.now("UTC"),
        updated_at=pendulum.now("UTC"),
    )
    session.add(user)
    session.commit()

    # Reading and converting to pendulum
    user = session.query(User).first()

    # The database returns a stdlib datetime; wrap it with pendulum
    created = pendulum.instance(user.created_at, tz="UTC")
    print(f"Created: {created.diff_for_humans()}")
    print(f"Created (ISO): {created.to_iso8601_string()}")

    # Querying with pendulum datetimes
    one_week_ago = pendulum.now("UTC").subtract(weeks=1)
    recent_users = (
        session.query(User)
        .filter(User.created_at >= one_week_ago)
        .all()
    )


# Custom type decorator for automatic conversion
from sqlalchemy import TypeDecorator


class PendulumDateTime(TypeDecorator):
    """SQLAlchemy column type that automatically converts to/from pendulum."""
    impl = DateTime(timezone=True)
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if isinstance(value, pendulum.DateTime):
            # Convert to stdlib datetime for the database driver
            return value.in_tz("UTC")
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            return pendulum.instance(value, tz="UTC")
        return value


class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(Integer, primary_key=True)
    action = Column(String(200))
    performed_at = Column(PendulumDateTime)  # auto-converts to pendulum on read
```

---

## Quick Reference

```python
import pendulum

# Creation
pendulum.now(tz="UTC")
pendulum.today(tz="UTC")
pendulum.tomorrow(tz="UTC")
pendulum.yesterday(tz="UTC")
pendulum.datetime(2026, 3, 15, tz="UTC")
pendulum.parse("2026-03-15T14:30:00")
pendulum.from_format("15/03/2026", "DD/MM/YYYY")
pendulum.from_timestamp(1773849600)
pendulum.instance(stdlib_datetime, tz="UTC")

# Arithmetic (all return new DateTime)
dt.add(years=1, months=2, days=3, hours=4, minutes=5, seconds=6)
dt.subtract(weeks=2, days=1)

# Differences
dt.diff(other)
dt.diff_for_humans()

# Boundaries
dt.start_of("day")     # also: year, month, week, hour, minute, second
dt.end_of("month")

# Formatting
dt.format("YYYY-MM-DD HH:mm:ss")
dt.to_iso8601_string()
dt.to_datetime_string()

# Timezone
dt.in_timezone("Asia/Tokyo")
dt.in_tz("UTC")

# Comparisons
dt.is_past()
dt.is_future()
dt.is_today()
dt.is_weekend()

# Durations and Periods
dur = pendulum.duration(days=30)
period = pendulum.period(start_dt, end_dt)
for dt in period.range("days"):
    pass
```
