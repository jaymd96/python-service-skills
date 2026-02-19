# py-scientist — Examples & Integration Patterns

> Part of the [scientist](../SKILL.md) skill. See SKILL.md for quick-start patterns.

## Table of Contents

- [Application Bootstrap](#application-bootstrap)
- [Business Logic Refactor](#business-logic-refactor)
- [Permission Model Migration](#permission-model-migration)
- [Algorithm Optimisation](#algorithm-optimisation)
- [Schema Migration with Transmute](#schema-migration-with-transmute)
- [Async Experiments in Cherry Handlers](#async-experiments-in-cherry-handlers)
- [Testing Experiments](#testing-experiments)
- [Querying Experiment Metrics](#querying-experiment-metrics)
- [Experiment Lifecycle](#experiment-lifecycle)
- [Gotchas and Anti-Patterns](#gotchas-and-anti-patterns)

---

## Application Bootstrap

Configure a default publisher once at application startup. All experiments inherit it.

### Cherry service with OTel + structured logging

```python
# app.py
from scientist import (
    set_default_publisher,
    new_composite_publisher,
    new_otel_publisher,
    new_log_publisher,
)

def create_app():
    # Configure experiment observability alongside OTel tracing
    set_default_publisher(
        new_composite_publisher(
            new_otel_publisher(meter_name="my-service"),
            new_log_publisher(),
        )
    )

    # ... rest of Cherry WitchcraftServer bootstrap
```

### Disable experiments in test environments

```python
# conftest.py
import pytest
from scientist import set_default_enabled, set_default_publisher, new_noop_publisher

@pytest.fixture(autouse=True)
def disable_experiments():
    """Experiments run deterministically in tests — no silent candidates."""
    set_default_enabled(False)
    set_default_publisher(new_noop_publisher())
    yield
    set_default_enabled(True)
```

### Enable with raise-on-mismatch in CI

```python
# conftest.py
import os
import pytest
from scientist import set_default_enabled

@pytest.fixture(autouse=True)
def strict_experiments():
    """In CI, experiments are enabled and mismatches fail the test."""
    if os.getenv("CI"):
        set_default_enabled(True)
    yield
```

```python
# In the experiment code, add raise_on_mismatches for CI safety:
exp = Experiment[Order]("order-total-v2")
exp.use(lambda: legacy_total(order))
exp.try_(lambda: new_total(order))
if os.getenv("CI"):
    exp.raise_on_mismatches()
result = exp.run()
```

---

## Business Logic Refactor

The most common use case: you've rewritten a function and want proof it behaves identically before switching.

```python
from scientist import Experiment

def calculate_shipping(order: Order) -> Money:
    exp = Experiment[Money]("shipping-calc-v2")
    exp.use(lambda: _legacy_shipping(order))
    exp.try_(lambda: _new_shipping(order))
    return exp.run()

def _legacy_shipping(order: Order) -> Money:
    # Original implementation — 200 lines, hard to reason about
    ...

def _new_shipping(order: Order) -> Money:
    # Rewrite — cleaner, but is it correct?
    ...
```

**Deployment sequence:**

1. Deploy with experiment enabled, 10% sample
2. Monitor `scientist.experiment.mismatches` counter for `shipping-calc-v2`
3. If zero mismatches over 48h at 10% → increase to 100%
4. If still zero over another 48h → flip: make new code the direct path
5. Remove experiment wrapper, delete legacy function

---

## Permission Model Migration

Migrating from a custom permission check to Casbin. Critical path — mistakes cause access control failures.

```python
from scientist import Experiment, comparator_from_func

def check_permission(user: User, resource: str, action: str) -> bool:
    exp = Experiment[bool]("casbin-authz")
    exp.use(lambda: _legacy_check(user, resource, action))
    exp.try_(lambda: _casbin_check(user, resource, action))

    # A mismatch where candidate DENIES but control ALLOWS is a
    # potential lockout — ignore during rollout, track separately
    exp.ignore(lambda r: (
        r.control.value is True
        and r.candidate.value is False
    ))

    return exp.run()
```

The ignore filter prevents lockouts from being counted as unexpected, but they're still published (with `ignored=True`) so you can query them:

```
scientist_experiment_total{experiment="casbin-authz",matched="false",ignored="true"}
```

---

## Algorithm Optimisation

Replacing a brute-force search with an indexed lookup. Results should be identical but floating-point or ordering differences might occur.

```python
from scientist import Experiment, comparator_from_func

def find_nearest(point: GeoPoint, radius_km: float) -> list[Location]:
    exp = Experiment[list[Location]]("geo-index-search")
    exp.use(lambda: _brute_force_search(point, radius_km))
    exp.try_(lambda: _rtree_search(point, radius_km))

    # Order may differ — compare as sets by ID
    exp.compare(comparator_from_func(
        lambda a, b: {loc.id for loc in a} == {loc.id for loc in b}
    ))

    # Only run on 5% — brute force is expensive
    exp.run_if(lambda: random.random() < 0.05)

    return exp.run()
```

---

## Schema Migration with Transmute

When Transmute is in the `RUNNING` state (dual-write active), use Scientist to verify that reads from the new schema match reads from the old schema.

```python
from scientist import Experiment, comparator_from_func

def get_user_profile(uid: str) -> UserProfile:
    exp = Experiment[UserProfile]("profile-new-schema")
    exp.use(lambda: _read_from_legacy_table(uid))
    exp.try_(lambda: _read_from_new_table(uid))

    # Compare on business fields, not metadata like updated_at
    exp.compare(comparator_from_func(
        lambda a, b: (a.name == b.name
                      and a.email == b.email
                      and a.role == b.role)
    ))

    return exp.run()
```

**When to use each:**

| Layer | Tool | Covers |
|-------|------|--------|
| Data writes | Transmute dual-write proxy | New schema receives all writes |
| Data reads | Scientist experiment | New schema returns correct data |
| Business logic | Scientist experiment | Rewritten logic behaves identically |

After the soak period, when Transmute moves to `AWAITING_FINALIZATION`:
1. Confirm Scientist mismatch rate is zero
2. Approve Transmute finalization (drops old schema)
3. Remove Scientist experiment wrapper
4. Delete legacy read function

---

## Async Experiments in Cherry Handlers

For async Cherry request handlers:

```python
from scientist import Experiment

async def handle_get_invoice(request: Request) -> Response:
    invoice_id = request.path_params["id"]

    exp = Experiment[Invoice]("invoice-fetch-v2")
    exp.use(lambda: legacy_fetch_invoice(invoice_id))
    exp.try_(lambda: new_fetch_invoice(invoice_id))

    invoice = await exp.async_run()
    return Response(json=invoice.to_dict())

async def legacy_fetch_invoice(invoice_id: str) -> Invoice:
    async with db.connection() as conn:
        row = await conn.fetch_one("SELECT ... FROM invoices_v1 WHERE id = $1", invoice_id)
        return Invoice.from_row(row)

async def new_fetch_invoice(invoice_id: str) -> Invoice:
    async with db.connection() as conn:
        row = await conn.fetch_one("SELECT ... FROM invoices_v2 WHERE id = $1", invoice_id)
        return Invoice.from_row(row)
```

Both control and candidate are awaited. Random execution order is preserved.

---

## Testing Experiments

### Unit testing the experiment itself

Use `raise_on_mismatches()` to turn mismatches into test failures:

```python
import pytest
from scientist import Experiment, ExperimentMismatchError

def test_new_shipping_matches_legacy():
    orders = [make_order(items=3), make_order(items=0), make_order(discount=True)]

    for order in orders:
        exp = Experiment[Money]("shipping-calc-v2")
        exp.use(lambda: _legacy_shipping(order))
        exp.try_(lambda: _new_shipping(order))
        exp.raise_on_mismatches()
        exp.run()  # raises ExperimentMismatchError if they differ
```

### Inspecting mismatches in tests

```python
from scientist import Experiment

def test_captures_mismatch_details():
    results = []

    class CapturingPublisher:
        def publish(self, result):
            results.append(result)

    exp = Experiment[int]("test-exp")
    exp.use(lambda: 42)
    exp.try_(lambda: 99)
    exp.publish(CapturingPublisher())
    exp.run()

    assert len(results) == 1
    assert results[0].mismatched
    assert results[0].control.value == 42
    assert results[0].candidate.value == 99
```

### Testing that experiments don't run when disabled

```python
from scientist import Experiment, set_default_enabled

def test_disabled_skips_candidate():
    candidate_ran = False

    def candidate():
        nonlocal candidate_ran
        candidate_ran = True
        return 99

    set_default_enabled(False)
    exp = Experiment[int]("disabled-test")
    exp.use(lambda: 42)
    exp.try_(candidate)
    result = exp.run()

    assert result == 42
    assert not candidate_ran
    set_default_enabled(True)
```

---

## Querying Experiment Metrics

Once `OTelPublisher` is configured and metrics flow to your backend (SigNoz, Prometheus, Grafana), use these queries to monitor experiments.

### PromQL: Mismatch rate

```promql
# Mismatch rate over last hour
rate(scientist_experiment_mismatches_total{experiment="shipping-calc-v2"}[1h])
/
rate(scientist_experiment_total_total{experiment="shipping-calc-v2"}[1h])
```

### PromQL: Zero mismatches confirmation

```promql
# Returns 1 if any mismatches occurred in the last 48h
scientist_experiment_mismatches_total{experiment="shipping-calc-v2"}
  - scientist_experiment_mismatches_total{experiment="shipping-calc-v2"} offset 48h
> 0
```

### PromQL: Candidate latency overhead

```promql
# p95 candidate duration vs control
histogram_quantile(0.95,
  rate(scientist_experiment_duration_bucket{experiment="shipping-calc-v2", behavior="candidate"}[1h])
)
/
histogram_quantile(0.95,
  rate(scientist_experiment_duration_bucket{experiment="shipping-calc-v2", behavior="control"}[1h])
)
```

A ratio > 2.0 means the candidate is significantly slower — consider sampling down with `run_if`.

### Grafana dashboard variables

```
Variable: experiment
Query: label_values(scientist_experiment_total_total, experiment)
```

---

## Experiment Lifecycle

The full lifecycle from creation to cleanup:

```
1. WRAP       Write experiment around the change point
                exp.use(old).try_(new)
                Deploy with run_if(10%)

2. OBSERVE    Monitor metrics for 24-48h
                Check mismatch rate = 0
                Check candidate latency acceptable

3. RAMP       Increase sampling
                run_if(50%) → run_if(100%)
                Monitor another 24-48h

4. FLIP       Make candidate the primary path
                Remove experiment wrapper
                New code runs directly

5. CLEAN UP   Delete legacy code
                Remove old function
                Update CHANGELOG
                Commit cleanup PR
```

### Decision criteria

| Metric | Threshold | Action |
|--------|-----------|--------|
| Mismatch rate | > 0 | Investigate — don't increase sampling |
| Mismatch rate | 0 for 48h at 100% | Safe to flip |
| Candidate p95 latency | > 2x control | Optimise candidate before flipping |
| Candidate error rate | > 0 | Fix candidate exceptions before flipping |

### When NOT to use Scientist

- **New features** with no legacy path to compare — there's no control
- **UI changes** — behaviour comparison doesn't apply to rendering
- **Write operations** — candidate would execute the write too (use feature flags instead)
- **Non-deterministic outputs** — random IDs, timestamps, etc. need custom comparators or `ignore` filters

---

## Gotchas and Anti-Patterns

### Side effects in candidates

The candidate runs for real. If it writes to a database, sends an email, or charges a credit card, those side effects happen.

```python
# WRONG — candidate also sends the email
exp.use(lambda: send_email_legacy(user))
exp.try_(lambda: send_email_new(user))

# RIGHT — compare the email content, don't send it
exp.use(lambda: build_email_legacy(user))
exp.try_(lambda: build_email_new(user))
```

### Capturing loop variables

```python
# WRONG — all experiments capture the final value of `order`
for order in orders:
    exp = Experiment[Money]("shipping")
    exp.use(lambda: calc(order))       # closes over loop var
    exp.try_(lambda: new_calc(order))
    exp.run()

# RIGHT — bind explicitly
for order in orders:
    exp = Experiment[Money]("shipping")
    exp.use(lambda o=order: calc(o))
    exp.try_(lambda o=order: new_calc(o))
    exp.run()
```

### Long-running experiments

Experiments add latency (both paths run). For hot paths:

1. Use `run_if` to sample
2. Monitor `scientist.experiment.duration` histogram
3. If candidate is slow, fix it before increasing sample rate

### Missing publisher configuration

If no publisher is set (instance or default), results go to `NoopPublisher` — you won't see mismatches. Always configure a publisher in application bootstrap.

### Exception comparison is type-only

```python
ValueError("bad input") vs ValueError("different message")  # → matched (same type)
ValueError("x") vs TypeError("x")                           # → mismatched (different type)
```

If exception messages matter, use a custom comparator with `ignore` filters.
