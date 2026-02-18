---
name: pluggy
description: Hook-based plugin system, battle-tested in pytest. Use when building plugin architectures, defining hook specifications and implementations, or adding extensibility points to applications. Triggers on plugin system, hooks, pluggy, hookspec, hookimpl, plugin architecture, extensibility.
---

# pluggy — Plugin System (v1.5.0)

## Quick Start

```bash
pip install pluggy
```

```python
import pluggy

hookspec = pluggy.HookspecMarker("myapp")
hookimpl = pluggy.HookimplMarker("myapp")

class MySpec:
    @hookspec
    def process(self, data):
        """Process data."""

class MyPlugin:
    @hookimpl
    def process(self, data):
        return data.upper()

pm = pluggy.PluginManager("myapp")
pm.add_hookspecs(MySpec)
pm.register(MyPlugin())
results = pm.hook.process(data="hello")  # ["HELLO"]
```

## Key Patterns

### Hook wrappers (before/after/error)
```python
class TimingPlugin:
    @hookimpl(wrapper=True)
    def process(self, data):
        start = time.time()
        result = yield  # other impls run here
        print(f"Took {time.time() - start}s")
        return result
```

### Entry point auto-discovery
```python
pm.load_setuptools_entrypoints("myapp")
# Loads plugins from: [project.entry-points.myapp] in pyproject.toml
```

## References

- **[api.md](references/api.md)** — PluginManager, HookspecMarker, HookimplMarker, registration API
- **[hooks.md](references/hooks.md)** — Hook specs/impls, firstresult, wrappers, ordering, historic hooks
- **[examples.md](references/examples.md)** — Complete examples, gotchas, plugin architecture patterns

## Grep Patterns

- `HookspecMarker|HookimplMarker` — Find hook definitions
- `PluginManager` — Find plugin manager setup
- `@hookspec|@hookimpl` — Find hook decorators
