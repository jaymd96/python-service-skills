---
name: croniter
description: Cron expression parser and datetime iterator. Use when parsing cron schedules, computing next/previous execution times, validating cron expressions, or iterating over cron-scheduled datetimes. Triggers on cron, cron expression, cron schedule, croniter, crontab, scheduled times.
---

# croniter — Cron Expression Iterator (v6.0.2)

## Quick Start

```bash
pip install croniter
```

```python
from croniter import croniter
from datetime import datetime

cron = croniter("0 9 * * MON-FRI", datetime(2025, 1, 1))
cron.get_next(datetime)  # next weekday at 9 AM
cron.get_next(datetime)  # following weekday at 9 AM
```

## Key Patterns

### Iterate forward/backward
```python
cron = croniter("*/15 * * * *", datetime(2025, 1, 1))
cron.get_next(datetime)   # 2025-01-01 00:15:00
cron.get_next(datetime)   # 2025-01-01 00:30:00
cron.get_prev(datetime)   # 2025-01-01 00:15:00 (reverses)
```

### Match and validate
```python
croniter.match("0 9 * * *", datetime(2025, 6, 15, 9, 0))   # True
croniter.is_valid("0 9 * * MON-FRI")  # True
```

### Get N occurrences
```python
import itertools
cron = croniter("0 0 1 * *", datetime(2025, 1, 1))
next_five = list(itertools.islice(cron.all_next(datetime), 5))
```

## References

- **[api.md](references/api.md)** — Full constructor API, iteration methods, match/validate, cron syntax (5/6/7-field), operators, and hash expressions
- **[examples.md](references/examples.md)** — Gotchas (day_or, stateful iterator, start time) and complete examples (scheduling, validation, seconds, backwards iteration)
