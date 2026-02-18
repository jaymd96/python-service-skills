---
name: python-dateutil
description: Powerful datetime extensions with flexible parsing and relative deltas. Use when parsing human-readable date strings, doing month/year arithmetic, generating recurrence rules, or handling timezones. Triggers on dateutil, date parsing, relativedelta, rrule, date arithmetic, fuzzy date parsing, month arithmetic.
---

# python-dateutil — Datetime Extensions (v2.9.0)

## Quick Start

```bash
pip install python-dateutil
```

```python
from dateutil import parser, relativedelta

dt = parser.parse("March 15, 2025 3:30pm")
next_month = dt + relativedelta.relativedelta(months=1)
```

## Key Patterns

### Flexible date parsing
```python
from dateutil import parser

parser.parse("2025-03-15")                # datetime(2025, 3, 15)
parser.parse("March 15, 2025")            # datetime(2025, 3, 15)
parser.parse("15/03/2025", dayfirst=True) # datetime(2025, 3, 15)
parser.parse("meeting on March 15 at 3pm", fuzzy=True)  # extracts date from text
```

### relativedelta (month/year arithmetic)
```python
from dateutil.relativedelta import relativedelta
from datetime import datetime

dt = datetime(2025, 1, 31)
dt + relativedelta(months=1)    # Feb 28 (handles month-end correctly)
dt + relativedelta(years=1, months=2, days=3)
```

### Recurrence rules (rrule)
```python
from dateutil.rrule import rrule, WEEKLY, MO, WE, FR

list(rrule(WEEKLY, count=10, byweekday=(MO, WE, FR),
           dtstart=datetime(2025, 1, 1)))
```

## References

- **[api.md](references/api.md)** — Full API for parser, relativedelta, rrule, rruleset, tz, and easter modules
- **[examples.md](references/examples.md)** — Gotchas (ambiguous dates, singular vs plural args) and complete examples (business days, age calc, timezones)
