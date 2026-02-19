---
name: scientist
description: Safe refactoring through controlled experiments using py-scientist — a typed Python port of GitHub's Scientist. Use when running old and new code paths in parallel, verifying refactors in production, comparing algorithm implementations, validating schema migrations, or measuring candidate behaviour without user impact. Triggers on scientist, experiment, controlled experiment, dual-path, safe refactoring, canary logic, mismatch, control candidate, py-scientist.
---

# py-scientist — Safe Refactoring Through Controlled Experiments (v0.1.0)

## Quick Start

```bash
pip install py-scientist            # zero required dependencies
pip install 'py-scientist[otel]'    # + OpenTelemetry metrics
pip install 'py-scientist[structlog]' # + structured logging
pip install 'py-scientist[all]'     # both
```

Minimal experiment:

```python
from scientist import Experiment

exp = Experiment[int]("pricing-calc")
exp.use(lambda: legacy_price(order))       # control — always returned
exp.try_(lambda: new_price(order))         # candidate — runs silently
result = exp.run()                         # returns legacy_price result
```

With observability:

```python
from scientist import (
    Experiment,
    new_otel_publisher,
    new_log_publisher,
    new_composite_publisher,
    set_default_publisher,
)

publisher = new_composite_publisher(
    new_otel_publisher(),    # metrics → OTLP → your backend
    new_log_publisher(),     # structured log on mismatch
)
set_default_publisher(publisher)

exp = Experiment[UserPermissions]("new-authz")
exp.use(lambda: legacy_check(user, resource))
exp.try_(lambda: new_check(user, resource))
perms = exp.run()  # always returns legacy result
```

## Key Patterns

### When to use Scientist

| Scenario | Control (old) | Candidate (new) |
|----------|--------------|-----------------|
| **Refactoring business logic** | Existing function | Rewritten function |
| **Permission model change** | Legacy policy check | New Casbin policy |
| **Algorithm optimisation** | Original algorithm | Optimised version |
| **Serialisation change** | Old cattrs converter | New converter |
| **Schema migration read path** | Read from old schema | Read from new schema |

Use Scientist when you need **proof of correctness before flipping** — not when you're adding new behaviour that has no old path to compare against.

### Gating: deterministic per-entity

Same customer always gets the same experience. Uses experiment name as salt so different experiments bucket independently:

```python
exp = Experiment[Money]("new-pricing")
exp.use(lambda: legacy_pricing(order))
exp.try_(lambda: new_pricing(order))
exp.run_if_entity(order.customer_id, percent=10)  # 10% of customers, deterministic
result = exp.run()
```

### Gating: group targeting

Beta testers and internal staff see the candidate:

```python
exp = Experiment[Dashboard]("new-dashboard")
exp.use(lambda: legacy_dashboard(user))
exp.try_(lambda: new_dashboard(user))
exp.run_if_group(allowed={"beta-testers", "internal"}, actual=user.groups)
result = exp.run()
```

### Gating: random per-request

For non-deterministic sampling on hot paths:

```python
exp = Experiment[Response]("new-serializer")
exp.use(lambda: old_serialize(payload))
exp.try_(lambda: new_serialize(payload))
exp.run_if_percent(10)  # 10% of requests, random
exp.run()
```

All gates also available as standalone factories via `from scientist.gates import entity_gate, group_gate, request_gate`.

### Custom comparison

Default uses `==`. Override for domain-specific equivalence:

```python
from scientist import Experiment, comparator_from_func

exp = Experiment[float]("revenue-calc")
exp.use(lambda: legacy_revenue(q))
exp.try_(lambda: new_revenue(q))
exp.compare(comparator_from_func(
    lambda control, candidate: abs(control - candidate) < 0.01
))
```

Or use the built-in `percent_difference_comparator`:

```python
from scientist import Experiment, percent_difference_comparator

exp = Experiment[float]("latency-estimate")
exp.use(lambda: old_estimate(req))
exp.try_(lambda: new_estimate(req))
exp.compare(percent_difference_comparator(threshold=0.05))  # 5% tolerance
```

### Ignoring known-acceptable mismatches

```python
exp = Experiment[dict]("user-profile")
exp.use(lambda: legacy_fetch(uid))
exp.try_(lambda: new_fetch(uid))
exp.ignore(lambda result: result.candidate.raised
           and isinstance(result.candidate.exception, TimeoutError))
```

### Async experiments

```python
exp = Experiment[UserProfile]("async-profile-fetch")
exp.use(lambda: legacy_fetch(uid))       # returns Awaitable
exp.try_(lambda: new_fetch(uid))
profile = await exp.async_run()
```

### Scientist + feature toggles (prove then flip)

```
Phase 1: PROVE  — Scientist runs both paths, returns control
                  exp.run_if_entity(customer_id, percent=10)
                  Ramp 10% → 50% → 100%, confirm zero mismatches

Phase 2: FLIP   — Remove experiment, use feature flag
                  if flags.new_pricing: return new(order)
                  Apollo flips the flag, zero downtime

Phase 3: CLEAN  — Remove flag, delete legacy code
```

See the [feature-toggle](../feature-toggle/SKILL.md) skill for the flag primitive.

### Scientist + Transmute (dual-path for data + logic)

```
Schema change?  → Transmute dual-write (data correctness)
Logic change?   → Scientist experiment (behaviour correctness)
Both?           → Transmute for data + Scientist for logic wrapping it
```

## References

- **[api.md](references/api.md)** — Complete API reference: Experiment, Observation, Result, Comparators, Publishers, context defaults
- **[examples.md](references/examples.md)** — Integration patterns, lifecycle management, OTel metric queries, cleanup workflow

## Grep Patterns

- `Experiment\[` — Find experiment definitions
- `\.use\(|\.try_\(` — Find control/candidate assignments
- `exp\.run\(\)|async_run\(\)` — Find experiment execution points
- `set_default_publisher|new_otel_publisher|new_log_publisher` — Find publisher configuration
- `run_if\(|run_if_entity\(|run_if_group\(|run_if_percent\(` — Find experiment gates
- `entity_gate\|group_gate\|request_gate` — Find standalone gate factories
- `raise_on_mismatches` — Find experiments that throw on mismatch
- `comparator_from_func|percent_difference` — Find custom comparators
