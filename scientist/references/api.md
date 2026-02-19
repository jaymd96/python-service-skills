# py-scientist — API Reference

> Part of the [scientist](../SKILL.md) skill. See SKILL.md for quick-start patterns.

## Table of Contents

- [Experiment](#experiment)
- [Observation](#observation)
- [Result](#result)
- [Comparators](#comparators)
- [Publishers](#publishers)
- [Gates](#gates)
- [Context Defaults](#context-defaults)
- [Errors](#errors)

---

## Experiment

```python
from scientist import Experiment
```

`Experiment[T]` is the core abstraction. Generic over the return type `T`. Uses a fluent builder pattern — all configuration methods return `self`.

### Constructor

```python
Experiment(name: str)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Identifies this experiment in logs and metrics |

### Configuration Methods

All return `self` for chaining.

#### `use(control: Callable[[], T]) -> Experiment[T]`

Set the control (old/trusted) behaviour. **Required.** The control's return value is always what `run()` returns to the caller.

#### `try_(candidate: Callable[[], T]) -> Experiment[T]`

Set the candidate (new/untrusted) behaviour. **Required.** Runs silently alongside control. Its return value is never exposed to the caller.

#### `compare(comparator: Comparator[T]) -> Experiment[T]`

Override the default equality comparator. See [Comparators](#comparators).

#### `publish(publisher: Publisher) -> Experiment[T]`

Set the publisher for this experiment. Overrides the context default.

#### `run_if(fn: Callable[[], bool]) -> Experiment[T]`

Gate function. Candidate only runs when `fn()` returns `True`. Use for sampling:

```python
exp.run_if(lambda: random.random() < 0.1)  # 10% sample
```

When gated off, only the control runs — zero overhead from the candidate.

#### `run_if_entity(entity_id: str, *, percent: float) -> Experiment[T]`

Deterministic per-entity gate. Uses `exp.name` as salt automatically. See [Gates](#gates).

```python
exp.run_if_entity(order.customer_id, percent=10)
```

#### `run_if_group(*, allowed: Collection[str], actual: Collection[str]) -> Experiment[T]`

Group membership gate. See [Gates](#gates).

```python
exp.run_if_group(allowed={"beta"}, actual=user.groups)
```

#### `run_if_percent(percent: float) -> Experiment[T]`

Random per-request gate. See [Gates](#gates).

```python
exp.run_if_percent(10)
```

#### `enabled(value: bool) -> Experiment[T]`

Explicitly enable or disable. When disabled, only control runs. Takes precedence over context default.

#### `ignore(fn: Callable[[Result[T]], bool]) -> Experiment[T]`

Add a filter for acceptable mismatches. Multiple filters can be chained — first returning `True` marks the mismatch as ignored.

```python
exp.ignore(lambda r: r.candidate.raised and isinstance(r.candidate.exception, TimeoutError))
exp.ignore(lambda r: r.control.value is None)  # both checked
```

Ignored mismatches are still published but flagged as `ignored=True`.

#### `before_run(fn: Callable[[], None]) -> Experiment[T]`

Add a pre-execution hook. Multiple hooks run in registration order. Exceptions propagate (not caught).

#### `clean(fn: Callable[[], None]) -> Experiment[T]`

Set cleanup function. Runs after the experiment completes, even on error. Only one — subsequent calls overwrite. Exceptions in cleanup are swallowed.

#### `raise_on_mismatches() -> Experiment[T]`

Enable raising `ExperimentMismatchError` on unexpected mismatches. Useful in test environments.

### Execution Methods

#### `run() -> T`

Run the experiment synchronously. Returns the control's value.

**Execution flow:**

1. Check enabled state (instance → context default)
2. Check `run_if` gate
3. Run `before_run` hooks
4. Execute control and candidate in **random order** (50/50 — avoids timing bias)
5. Compare observations using comparator
6. Apply ignore filters to mismatches
7. Publish result (errors swallowed)
8. If `raise_on_mismatches` and unexpected mismatch → raise
9. Run cleanup (always, errors swallowed)
10. Return control value (or re-raise control exception)

When disabled or gated off: only control runs, no publishing.

#### `async_run() -> T`

Async variant. Same flow as `run()` but awaits async callables.

```python
exp.use(lambda: fetch_legacy(uid))   # returns Awaitable[T]
exp.try_(lambda: fetch_new(uid))
result = await exp.async_run()
```

---

## Observation

```python
from scientist import Observation
from scientist.observation import observe, async_observe
```

Frozen dataclass capturing a single behaviour execution.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | `"control"` or `"candidate"` |
| `value` | `T \| None` | Return value, `None` if exception raised |
| `exception` | `BaseException \| None` | Captured exception, `None` if succeeded |
| `duration_seconds` | `float` | Wall-clock time (`time.perf_counter`) |
| `cpu_time_seconds` | `float` | CPU time (`time.process_time`) |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `raised` | `bool` | Whether the behaviour raised an exception |
| `value_or_raise` | `T` | Returns value or re-raises captured exception |

### `equivalent_to(other: Observation[T]) -> bool`

Returns `True` if both succeeded with equal values, or both raised the same exception type.

### Factory Functions

```python
def observe(name: str, behavior: Callable[[], T]) -> Observation[T]
```

Execute a synchronous callable and capture timing + result/exception.

```python
async def async_observe(name: str, behavior: Callable[[], Awaitable[T]]) -> Observation[T]
```

Async variant.

---

## Result

```python
from scientist import Result
```

Frozen dataclass aggregating experiment outcome.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `experiment_name` | `str` | Name from `Experiment(name)` |
| `control` | `Observation[T]` | Control observation |
| `candidate` | `Observation[T]` | Candidate observation |
| `matched` | `bool` | Whether comparator reported equivalence |
| `ignored` | `bool` | Whether a mismatch was filtered by `ignore()` |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `mismatched` | `bool` | `not self.matched` |
| `ignored_mismatch` | `bool` | `self.mismatched and self.ignored` |
| `unexpected_mismatch` | `bool` | `self.mismatched and not self.ignored` |

---

## Comparators

```python
from scientist import (
    DefaultComparator,
    CallableComparator,
    comparator_from_func,
    percent_difference_comparator,
    set_comparator,
)
```

### Comparator Protocol

```python
class Comparator(Protocol[T]):
    def compare(self, control: T, candidate: T) -> bool: ...
```

Runtime-checkable. Implement this protocol for custom comparators.

### Built-in Comparators

#### `DefaultComparator[T]`

Uses `==`. This is the default when no comparator is set.

#### `comparator_from_func(fn: Callable[[T, T], bool]) -> Comparator[T]`

Wrap any comparison function:

```python
comparator_from_func(lambda a, b: a.id == b.id)
```

#### `percent_difference_comparator(threshold: float) -> Comparator[float]`

For numeric values. Returns `True` when the percentage difference is within threshold.

| Parameter | Type | Description |
|-----------|------|-------------|
| `threshold` | `float` | Maximum acceptable difference (0.0–1.0) |

```python
percent_difference_comparator(0.05)  # 5% tolerance
```

Edge cases: both zero → `True`, one zero → `False`.

#### `set_comparator() -> Comparator[set[object]]`

Set equality comparison.

### Comparison Logic

When the experiment compares observations:

1. **Both raised** → compare exception types (`type(a) == type(b)`)
2. **Both succeeded** → call `comparator.compare(control.value, candidate.value)`
3. **One raised, one didn't** → `False` (always a mismatch)

---

## Publishers

```python
from scientist import (
    NoopPublisher,
    LogPublisher,
    OTelPublisher,
    CompositePublisher,
    new_noop_publisher,
    new_log_publisher,
    new_otel_publisher,
    new_composite_publisher,
)
```

### Publisher Protocol

```python
class Publisher(Protocol):
    def publish(self, result: object) -> None: ...
```

Runtime-checkable. Publisher errors are always swallowed — they never affect the control return value.

### NoopPublisher

Does nothing. Used as fallback when no publisher is configured.

```python
publisher = new_noop_publisher()
```

### LogPublisher

Logs via structlog. Gracefully degrades to no-op if structlog is not installed.

```python
publisher = new_log_publisher()
```

| Outcome | Log level | Message |
|---------|-----------|---------|
| Matched | `info` | `"experiment matched"` |
| Ignored mismatch | `info` | `"experiment mismatched (ignored)"` |
| Unexpected mismatch | `warning` | `"experiment mismatched"` |

Log fields: `experiment`, `control_value`, `candidate_value`, `control_duration`, `candidate_duration`, `control_raised`, `candidate_raised`.

### OTelPublisher

Records OpenTelemetry metrics. Lazy-initializes on first `publish()` call. Gracefully degrades if `opentelemetry-api` is not installed.

```python
publisher = new_otel_publisher()                      # default meter: "scientist"
publisher = new_otel_publisher(meter_name="my-app")   # custom meter name
```

| Metric | Type | Attributes | When |
|--------|------|------------|------|
| `scientist.experiment.total` | Counter | `experiment`, `matched`, `ignored` | Every run |
| `scientist.experiment.mismatches` | Counter | `experiment` | Unexpected mismatch only |
| `scientist.experiment.duration` | Histogram (seconds) | `experiment`, `behavior` | Both control and candidate |

On unexpected mismatch with an active span, adds a span event `"scientist.mismatch"` with `experiment`, `control.value`, `candidate.value`.

### CompositePublisher

Forwards to all contained publishers. Each publisher's errors are caught independently — one failure doesn't stop the others.

```python
publisher = new_composite_publisher(
    new_otel_publisher(),
    new_log_publisher(),
)
```

---

## Gates

```python
from scientist import entity_gate, group_gate, request_gate
# or
from scientist.gates import entity_gate, group_gate, request_gate
```

Factory functions that return `Callable[[], bool]` for use with `Experiment.run_if()`. Also available as syntactic sugar directly on Experiment.

### `entity_gate(entity_id, *, percent, salt="")` — Deterministic per-entity

Same entity always gets the same True/False for a given salt. Uses SHA-256 hash of `(salt, entity_id)` mod 100.

| Parameter | Type | Description |
|-----------|------|-------------|
| `entity_id` | `str` | Stable identifier (customer ID, account ID) |
| `percent` | `float` | Percentage of entities to include (0–100) |
| `salt` | `str` | Differentiator per experiment (use experiment name) |

```python
gate = entity_gate("customer-123", percent=10, salt="new-pricing")
gate()  # always the same bool for this customer + salt
```

**Syntactic sugar on Experiment** — uses `exp.name` as salt automatically:

```python
exp.run_if_entity(order.customer_id, percent=10)
```

### `group_gate(*, allowed, actual)` — Group membership

Returns True when the entity belongs to at least one allowed group.

| Parameter | Type | Description |
|-----------|------|-------------|
| `allowed` | `Collection[str]` | Groups that should see the candidate |
| `actual` | `Collection[str]` | Groups the current entity belongs to |

```python
gate = group_gate(allowed={"beta", "internal"}, actual=user.groups)
gate()  # True if user is in beta or internal
```

**Syntactic sugar:**

```python
exp.run_if_group(allowed={"beta", "internal"}, actual=user.groups)
```

### `request_gate(*, percent)` — Random per-request

Each invocation has an independent chance of returning True. Non-deterministic.

| Parameter | Type | Description |
|-----------|------|-------------|
| `percent` | `float` | Percentage of requests to include (0–100) |

```python
gate = request_gate(percent=10)
gate()  # ~10% chance of True, independent each call
```

**Syntactic sugar:**

```python
exp.run_if_percent(10)
```

---

## Context Defaults

```python
from scientist import (
    set_default_publisher,
    get_default_publisher,
    set_default_enabled,
    get_default_enabled,
)
```

Thread-safe via `contextvars.ContextVar`.

### `set_default_publisher(publisher: Publisher | None) -> None`

Set the default publisher for all experiments that don't have an explicit publisher. Pass `None` to clear.

### `get_default_publisher() -> Publisher | None`

Returns the current default, or `None`.

### `set_default_enabled(enabled: bool) -> None`

Control whether experiments run by default. Defaults to `True`.

```python
set_default_enabled(False)  # disable all experiments globally
```

### `get_default_enabled() -> bool`

Returns current state. Defaults to `True`.

### Precedence

| Setting | Checked first | Then |
|---------|--------------|------|
| Publisher | `exp.publish(p)` | `get_default_publisher()` → `NoopPublisher` |
| Enabled | `exp.enabled(v)` | `get_default_enabled()` (default `True`) |

---

## Errors

```python
from scientist import ExperimentMismatchError
```

### ExperimentMismatchError

```python
class ExperimentMismatchError(Generic[T], Exception):
    result: Result[T]
```

Raised only when `exp.raise_on_mismatches()` is set and an unexpected (non-ignored) mismatch occurs.

```python
try:
    exp.raise_on_mismatches()
    result = exp.run()
except ExperimentMismatchError as e:
    print(e.result.control.value)
    print(e.result.candidate.value)
```

Useful in test suites to catch mismatches as failures.
