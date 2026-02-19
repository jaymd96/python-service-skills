---
name: feature-toggle
description: Simple feature flags backed by Cherry's Refreshable configuration, toggled via runtime YAML or environment variables and propagated through Apollo. Use when enabling or disabling features at runtime, gating new behaviour behind flags, rolling out changes to specific entity groups via Scientist, or cleaning up flags after rollout. Triggers on feature flag, feature toggle, feature gate, feature switch, runtime toggle, config flag, refreshable config, rollout, canary.
---

# Feature Toggles — Refreshable Config Flags for Cherry Services (v1.0.0)

## Quick Start

Add flags to your runtime config:

```yaml
# var/conf/runtime.yml
feature_flags:
  new_checkout: true
  dark_mode: false
  experimental_search: false
```

Define the dynamic flags container:

```python
# config.py
from __future__ import annotations
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
        """String-based access for hyphenated or dynamic flag names."""
        return self._flags.get(name, False)

    def __repr__(self) -> str:
        enabled = [k for k, v in self._flags.items() if v]
        return f"FeatureFlags({', '.join(enabled) or 'none enabled'})"
```

Wire into Cherry's `RuntimeConfig`:

```python
from dataclasses import dataclass, field

@dataclass
class RuntimeConfig:
    # ... existing Cherry fields ...
    feature_flags: FeatureFlags = field(default_factory=FeatureFlags)
```

Use in handlers:

```python
def checkout(order: Order) -> Response:
    flags = ctx.runtime_config.get().feature_flags
    if flags.new_checkout:
        return new_checkout(order)
    return legacy_checkout(order)
```

Apollo changes the YAML → Cherry's `FileRefreshable` detects the change → `RuntimeConfig` reloads → flag flips → next request picks it up. Zero downtime, no redeploy.

## Key Patterns

### Any key, no code changes to add a flag

`FeatureFlags` accepts any key. Adding a new flag is a config-only change:

```yaml
# Just add a line — no Python change needed
feature_flags:
  new_checkout: true
  dark_mode: false
  beta_reporting: true   # new flag, available immediately
```

```python
flags.beta_reporting  # True — no FeatureFlags class change needed
flags.nonexistent     # False — unknown flags default to off
```

### Combining with Scientist for safe rollout

Feature flags are the **on/off switch**. Scientist is the **proof of correctness**. Together they form the full rollout lifecycle:

```python
from scientist import Experiment

def calculate_price(order: Order) -> Money:
    # Phase 1: PROVE — Scientist runs both paths, returns control
    exp = Experiment[Money]("new-pricing")
    exp.use(lambda: legacy_pricing(order))
    exp.try_(lambda: new_pricing(order))
    exp.run_if_entity(order.customer_id, percent=10)
    return exp.run()

# Phase 2: FLIP — after zero mismatches, use the flag
def calculate_price(order: Order) -> Money:
    if ctx.runtime_config.get().feature_flags.new_pricing:
        return new_pricing(order)
    return legacy_pricing(order)

# Phase 3: CLEAN UP — remove flag, delete legacy code
def calculate_price(order: Order) -> Money:
    return new_pricing(order)
```

### Deterministic per-entity gating with Scientist

Same customer always gets the same experience:

```python
exp = Experiment[Dashboard]("new-dashboard")
exp.use(lambda: legacy_dashboard(user))
exp.try_(lambda: new_dashboard(user))
exp.run_if_entity(user.id, percent=20)  # 20% of users, deterministic
result = exp.run()
```

### Group-based targeting with Scientist

Beta testers and internal staff see the candidate:

```python
exp = Experiment[Response]("new-search")
exp.use(lambda: legacy_search(query))
exp.try_(lambda: new_search(query))
exp.run_if_group(allowed={"beta-testers", "internal"}, actual=user.groups)
result = exp.run()
```

### The propagation chain

```
Apollo config change
  → Helm values.yaml update
    → Kubernetes ConfigMap mounted at var/conf/runtime.yml
      → Cherry FileRefreshable detects file change
        → RuntimeConfig reloads with new FeatureFlags
          → ctx.runtime_config.get().feature_flags.X returns new value
```

## References

- **[api.md](references/api.md)** — FeatureFlags class, RuntimeConfig wiring, Refreshable integration, YAML loader
- **[examples.md](references/examples.md)** — Rollout lifecycle, Scientist composition, Apollo propagation, cleanup patterns

## Grep Patterns

- `feature_flags` — Find flag access in handlers
- `FeatureFlags` — Find the flags class definition
- `is_enabled\(` — Find string-based flag checks
- `runtime_config\.get\(\)` — Find runtime config reads
- `run_if_entity\|run_if_group\|run_if_percent` — Find Scientist gate usage
