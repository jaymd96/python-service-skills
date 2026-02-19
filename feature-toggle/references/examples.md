# Feature Toggle — Examples & Integration Patterns

> Part of the [feature-toggle](../SKILL.md) skill. See SKILL.md for quick-start patterns.

## Table of Contents

- [Complete Service Setup](#complete-service-setup)
- [Rollout Lifecycle](#rollout-lifecycle)
- [Side-Effect Changes](#side-effect-changes)
- [Apollo Propagation](#apollo-propagation)
- [Testing with Flags](#testing-with-flags)
- [Flag Cleanup](#flag-cleanup)
- [Anti-Patterns](#anti-patterns)

---

## Complete Service Setup

Full working example: Cherry service with feature flags + Scientist.

### Config module

```python
# my_service/config.py
from __future__ import annotations

import yaml
from dataclasses import dataclass, field


class FeatureFlags:
    """Dynamic feature flag container. Any key defaults to False."""

    def __init__(self, **flags: bool) -> None:
        self._flags = flags

    def __getattr__(self, name: str) -> bool:
        try:
            return self._flags[name]
        except KeyError:
            return False

    def is_enabled(self, name: str) -> bool:
        return self._flags.get(name, False)

    def __repr__(self) -> str:
        enabled = [k for k, v in self._flags.items() if v]
        return f"FeatureFlags({', '.join(enabled) or 'none enabled'})"


@dataclass
class RuntimeConfig:
    feature_flags: FeatureFlags = field(default_factory=FeatureFlags)


def load_runtime_config(file_path: str) -> RuntimeConfig:
    with open(file_path) as f:
        data = yaml.safe_load(f) or {}
    flags_data = data.pop("feature_flags", {})
    return RuntimeConfig(feature_flags=FeatureFlags(**flags_data))
```

### Application bootstrap

```python
# my_service/app.py
from py_witchcraft_server.config.refreshable import new_file_refreshable
from py_witchcraft_server.server import WitchcraftServer
from scientist import set_default_publisher, new_composite_publisher, new_otel_publisher, new_log_publisher

from .config import load_runtime_config

def create_app() -> WitchcraftServer:
    # Scientist observability
    set_default_publisher(
        new_composite_publisher(
            new_otel_publisher(meter_name="my-service"),
            new_log_publisher(),
        )
    )

    server = WitchcraftServer("my-service")
    server.with_runtime_config_refreshable(
        new_file_refreshable("var/conf/runtime.yml", load_runtime_config)
    )
    return server
```

### Runtime config

```yaml
# var/conf/runtime.yml
feature_flags:
  new_pricing: false
  beta_search: false
```

---

## Rollout Lifecycle

The three-phase pattern for safe feature rollout.

### Phase 1: PROVE with Scientist

Run old and new code in parallel. Control always wins. Scientist measures correctness.

```python
from scientist import Experiment

def calculate_price(order: Order, ctx: RequestContext) -> Money:
    exp = Experiment[Money]("new-pricing")
    exp.use(lambda: legacy_pricing(order))
    exp.try_(lambda: new_pricing(order))
    exp.run_if_entity(order.customer_id, percent=10)
    return exp.run()
```

Monitor:
- `scientist.experiment.mismatches{experiment="new-pricing"}` should be 0
- `scientist.experiment.duration{behavior="candidate"}` should be acceptable

Ramp: change `percent=10` → `50` → `100` via code deploys. At each stage, confirm zero mismatches for 24-48h.

### Phase 2: FLIP with feature flag

After proving correctness at 100%, remove the experiment and use a simple flag:

```python
def calculate_price(order: Order, ctx: RequestContext) -> Money:
    if ctx.runtime_config.get().feature_flags.new_pricing:
        return new_pricing(order)
    return legacy_pricing(order)
```

Apollo flips `new_pricing: true` in the config. Instant, no redeploy.

### Phase 3: CLEAN UP

Remove the flag, delete the legacy code:

```python
def calculate_price(order: Order, ctx: RequestContext) -> Money:
    return new_pricing(order)
```

Remove `new_pricing` from `runtime.yml`. Update CHANGELOG.

---

## Side-Effect Changes

When you can't use Scientist (candidate has side effects), go straight to flags.

### Email template change

```python
def send_receipt(order: Order, ctx: RequestContext) -> None:
    flags = ctx.runtime_config.get().feature_flags
    if flags.new_receipt_template:
        send_new_receipt(order)
    else:
        send_legacy_receipt(order)
```

Can't run both — both would send an email. Use the flag directly, monitor error rates.

### Payment flow change

```python
def process_payment(order: Order, ctx: RequestContext) -> PaymentResult:
    flags = ctx.runtime_config.get().feature_flags
    if flags.stripe_v2:
        return process_stripe_v2(order)
    return process_stripe_v1(order)
```

For high-risk side-effect changes, consider running in shadow mode first by logging what the new path *would* do without executing it.

---

## Apollo Propagation

How a flag change travels from Apollo to your handler.

### Step 1: Update Helm values

```yaml
# values.yaml (in Helm chart)
config:
  runtime:
    feature_flags:
      new_pricing: true    # was false
```

### Step 2: Apollo creates config_update plan

```bash
apollo entity update my-service-prod --values values.yaml
```

The OrchestrationEngine creates a `config_update` plan (no version change, just config).

### Step 3: Spoke agent executes

```bash
# SpokeAgent runs:
helm upgrade my-service-prod ./chart --values values.yaml
```

### Step 4: Kubernetes updates ConfigMap

The ConfigMap backing `var/conf/runtime.yml` is updated. Kubernetes propagates the change to the mounted volume.

### Step 5: Cherry detects and reloads

FileRefreshable's watchdog observer fires `on_modified`. The loader function re-reads the YAML. `Refreshable.set()` stores the new config and notifies subscribers.

### Step 6: Next request reads new value

```python
# This now returns True
ctx.runtime_config.get().feature_flags.new_pricing
```

**Latency:** Typically < 2 seconds from ConfigMap update to handler seeing the new value.

---

## Testing with Flags

### Override flags in tests

```python
import pytest
from my_service.config import FeatureFlags, RuntimeConfig
from py_witchcraft_server.config.refreshable import new_refreshable


@pytest.fixture
def feature_flags():
    """Provides controllable feature flags for testing."""
    flags = FeatureFlags(new_pricing=True, dark_mode=False)
    config = RuntimeConfig(feature_flags=flags)
    return new_refreshable(config)


def test_new_pricing_path(feature_flags):
    # feature_flags.get().feature_flags.new_pricing is True
    result = calculate_price(order, ctx_with(feature_flags))
    assert result == expected_new_price


def test_legacy_pricing_path():
    flags = FeatureFlags()  # all flags off by default
    config = RuntimeConfig(feature_flags=flags)
    refreshable = new_refreshable(config)
    result = calculate_price(order, ctx_with(refreshable))
    assert result == expected_legacy_price
```

### Test flag toggle mid-request

```python
def test_flag_change_takes_effect():
    flags_off = FeatureFlags(new_pricing=False)
    config = RuntimeConfig(feature_flags=flags_off)
    refreshable = new_refreshable(config)

    assert not refreshable.get().feature_flags.new_pricing

    # Simulate config reload
    flags_on = FeatureFlags(new_pricing=True)
    refreshable.set(RuntimeConfig(feature_flags=flags_on))

    assert refreshable.get().feature_flags.new_pricing
```

### Test unknown flags default to False

```python
def test_unknown_flags_are_off():
    flags = FeatureFlags(known_flag=True)
    assert flags.known_flag is True
    assert flags.completely_unknown is False
```

---

## Flag Cleanup

Flags are temporary. Every flag should have a cleanup plan.

### When to clean up

| Signal | Action |
|--------|--------|
| Flag has been `true` in all environments for > 7 days | Remove flag, delete old code |
| Scientist showed 0 mismatches at 100% for 48h | Safe to flip flag |
| Flag was a rollback mechanism and rollback window has passed | Remove flag |

### Cleanup checklist

1. Remove flag check from handler code
2. Delete the legacy function
3. Remove flag from `runtime.yml` in all environments
4. Remove flag from Helm values
5. Add `### Removed` entry to CHANGELOG
6. Create cleanup PR

### Finding stale flags

```bash
# Flags defined in config
grep -r "feature_flags:" var/conf/ values.yaml

# Flags referenced in code
grep -rn "feature_flags\.\|is_enabled(" src/

# Compare — flags in config but not in code are dead
```

---

## Anti-Patterns

### Flag nesting

```python
# WRONG — combinatorial explosion
if flags.new_checkout:
    if flags.new_payment:
        if flags.new_receipt:
            ...
```

Each flag should control one thing. If features depend on each other, make one flag that enables the whole flow.

### Flags for permanent config

```python
# WRONG — this is configuration, not a feature toggle
if flags.max_retries_5:
    retries = 5
else:
    retries = 3
```

Use dynaconf settings for permanent configuration. Feature flags are temporary.

### Testing only the happy path

```python
# Test BOTH paths
def test_with_flag_on(): ...
def test_with_flag_off(): ...
```

The legacy path is still live until the flag is removed. Both paths need test coverage.

### Forgetting cleanup

Every flag introduced should have a CHANGELOG entry for both the addition (`### Added`) and the eventual removal (`### Removed`). If a flag has been on for 30+ days with no cleanup plan, it's tech debt.
