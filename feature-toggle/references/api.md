# Feature Toggle — API Reference

> Part of the [feature-toggle](../SKILL.md) skill. See SKILL.md for quick-start patterns.

## Table of Contents

- [FeatureFlags](#featureflags)
- [RuntimeConfig Integration](#runtimeconfig-integration)
- [YAML Loading](#yaml-loading)
- [Refreshable Wiring](#refreshable-wiring)
- [Scientist Gate Helpers](#scientist-gate-helpers)

---

## FeatureFlags

Dynamic container for boolean feature flags. Any key can be added via config without changing the class.

```python
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

### Attribute Access

```python
flags = FeatureFlags(new_checkout=True, dark_mode=False)

flags.new_checkout    # True
flags.dark_mode       # False
flags.nonexistent     # False (unknown flags default to off)
```

### String Access

For flag names with hyphens or when the name is dynamic:

```python
flags.is_enabled("new-checkout")      # requires string lookup
flags.is_enabled(flag_name_from_db)   # dynamic flag names
```

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| `**kwargs` constructor | YAML loader passes dict as kwargs — no custom deserialisation needed |
| Unknown keys return `False` | Safe default — missing flag means feature is off |
| No `__setattr__` | Flags are immutable per config reload. New values arrive via Refreshable |
| No validation | Any string key accepted. Typos fail silently (return False). Use grep patterns to audit |

---

## RuntimeConfig Integration

Add `FeatureFlags` to Cherry's `RuntimeConfig`:

```python
from dataclasses import dataclass, field

@dataclass
class RuntimeConfig:
    diagnostics: DiagnosticsConfig = field(default_factory=DiagnosticsConfig)
    health_checks: HealthChecksConfig = field(default_factory=HealthChecksConfig)
    audit: AuditConfig | None = None
    logging: LoggerConfig | None = None
    feature_flags: FeatureFlags = field(default_factory=FeatureFlags)
```

### Custom YAML Loader

Cherry's `yaml_loader` expects dataclasses. Since `FeatureFlags` uses `**kwargs`, provide a custom loader:

```python
import yaml

def runtime_config_loader(file_path: str) -> RuntimeConfig:
    """Load RuntimeConfig from YAML, handling FeatureFlags specially."""
    with open(file_path) as f:
        data = yaml.safe_load(f) or {}

    flags_data = data.pop("feature_flags", {})
    feature_flags = FeatureFlags(**flags_data)

    return RuntimeConfig(
        **{k: v for k, v in data.items() if k != "feature_flags"},
        feature_flags=feature_flags,
    )
```

### Wire to Server

```python
from py_witchcraft_server.config.refreshable import new_file_refreshable

server = WitchcraftServer("my-service")
server.with_runtime_config_refreshable(
    new_file_refreshable("var/conf/runtime.yml", runtime_config_loader)
)
```

---

## YAML Loading

### Config File Format

```yaml
# var/conf/runtime.yml
diagnostics:
  debug_shared_secret_file: "var/conf/debug-secret"

health_checks:
  shared_secret: "health-check-secret"

logging:
  level: INFO

feature_flags:
  new_checkout: true
  dark_mode: false
  experimental_search: true
  beta_reporting: false
```

### Adding a New Flag

Add a line to the YAML. No Python code changes required:

```yaml
feature_flags:
  new_checkout: true
  dark_mode: false
  my_new_feature: true   # available immediately after config reload
```

### Environment Variable Override via dynaconf

When using dynaconf for config management, flags can also be set via environment:

```bash
export MYAPP_FEATURE_FLAGS__NEW_CHECKOUT=true
export MYAPP_FEATURE_FLAGS__DARK_MODE=false
```

dynaconf's double-underscore notation maps to nested config keys.

---

## Refreshable Wiring

### How Config Reload Works

```
1. Apollo pushes new Helm values
2. Kubernetes updates ConfigMap → mounted file changes
3. Cherry's FileRefreshable (watchdog) detects modification
4. Loader function re-reads YAML
5. Refreshable.set() stores new RuntimeConfig
6. Subscribers notified (if any)
7. Next ctx.runtime_config.get() returns updated config
```

### Reading Flags in Handlers

```python
# Direct attribute access (most common)
def handle_request(ctx: RequestContext) -> Response:
    flags = ctx.runtime_config.get().feature_flags
    if flags.new_checkout:
        return new_checkout_handler(ctx)
    return legacy_checkout_handler(ctx)
```

### Subscribing to Flag Changes

For cases where you need to react to a flag change (pre-warm a cache, reconnect, etc.):

```python
def on_config_change(new_config: RuntimeConfig) -> None:
    if new_config.feature_flags.experimental_search:
        warm_search_index()

ctx.runtime_config.subscribe(on_config_change)
```

### Deriving Values with map()

```python
search_enabled = ctx.runtime_config.map(
    lambda c: c.feature_flags.experimental_search
)
```

---

## Scientist Gate Helpers

py-scientist provides syntactic sugar for gating experiments. These complement feature flags — Scientist proves correctness, flags control the final on/off.

### `run_if_entity(entity_id, *, percent)` — Deterministic per-entity

```python
from scientist import Experiment

exp = Experiment[Money]("new-pricing")
exp.use(lambda: legacy(order))
exp.try_(lambda: new(order))
exp.run_if_entity(order.customer_id, percent=10)
result = exp.run()
```

Uses SHA-256 hash of `(experiment_name, entity_id)` mod 100. Same customer always gets the same result for the same experiment. Different experiments bucket independently.

### `run_if_group(*, allowed, actual)` — Group membership

```python
exp.run_if_group(allowed={"beta", "internal"}, actual=user.groups)
```

Candidate runs when the entity belongs to at least one allowed group.

### `run_if_percent(percent)` — Random per-request

```python
exp.run_if_percent(10)  # 10% of requests, non-deterministic
```

Each invocation is independent — same entity may get different results across requests.

### Standalone Gate Functions

The sugar methods delegate to factory functions in `scientist.gates`:

```python
from scientist.gates import entity_gate, group_gate, request_gate

# Composable — use with exp.run_if() directly
exp.run_if(entity_gate("user-123", percent=10, salt="my-exp"))
exp.run_if(group_gate(allowed={"beta"}, actual=user.groups))
exp.run_if(request_gate(percent=5))
```

The standalone functions accept an explicit `salt` parameter. The `run_if_entity` sugar uses `exp.name` as salt automatically.
