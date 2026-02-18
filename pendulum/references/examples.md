# Pendulum — Examples & Gotchas

> Part of the pendulum skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [Pendulum 2.x vs 3.x Breaking Changes](#pendulum-2x-vs-3x-breaking-changes)
  - [Serialization / JSON](#serialization--json)
  - [Interaction with SQLAlchemy](#interaction-with-sqlalchemy)
  - [Interaction with attrs / cattrs](#interaction-with-attrs--cattrs)
  - [Immutability -- add/subtract Return New Objects](#immutability----addsubtract-return-new-objects)
  - [Performance vs. stdlib datetime](#performance-vs-stdlib-datetime)
  - [Pickle Support](#pickle-support)
  - [Testing and Time Mocking](#testing-and-time-mocking)
- [Complete Code Examples](#complete-code-examples)
  - [Basic DateTime Creation and Manipulation](#basic-datetime-creation-and-manipulation)
  - [Timezone Conversions](#timezone-conversions)
  - [Date Arithmetic](#date-arithmetic)
  - [Human-Readable Diffs](#human-readable-diffs)
  - [Iterating Over Date Ranges](#iterating-over-date-ranges)
  - [Formatting and Parsing](#formatting-and-parsing)
  - [Integration with SQLAlchemy Columns](#integration-with-sqlalchemy-columns)

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
