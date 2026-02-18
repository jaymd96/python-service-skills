# pluggy — API Reference

> Part of the pluggy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API Reference](#core-api-reference)
- [Entry Points Integration](#entry-points-integration)
- [API Quick Reference Table](#api-quick-reference-table)

## Core API Reference

```
# Pluggy API Reference
# Version: 1.5.0 | Python: 3.8+

## Marker Decorators

# Create a hook specification marker for your project
import pluggy
hookspec = pluggy.HookspecMarker("myproject")
hookimpl = pluggy.HookimplMarker("myproject")

## HookspecMarker(project_name)
# Returns a decorator that marks functions as hook specifications.
# The project_name ties this spec to a specific plugin manager.

hookspec = pluggy.HookspecMarker("myproject")

class MySpec:
    @hookspec
    def my_hook(self, arg1, arg2):
        """Docstring describes what this extension point does."""

    @hookspec(firstresult=True)
    def my_single_hook(self, query):
        """Only the first non-None result is returned."""

    @hookspec(historic=True)
    def my_historic_hook(self, config):
        """Replays to plugins registered after the initial call."""

## HookimplMarker(project_name)
# Returns a decorator that marks functions as hook implementations.

hookimpl = pluggy.HookimplMarker("myproject")

class MyPlugin:
    @hookimpl
    def my_hook(self, arg1, arg2):
        return do_something(arg1, arg2)

    @hookimpl(tryfirst=True)
    def my_hook(self, arg1, arg2):
        """Run before other implementations."""

    @hookimpl(trylast=True)
    def my_hook(self, arg1, arg2):
        """Run after other implementations."""

    @hookimpl(wrapper=True)
    def my_hook(self, arg1, arg2):
        """Wrap other implementations (yield-based)."""
        # Before
        result = yield
        # After (result is the outcome from inner hooks)
        return modified_result

    @hookimpl(specname="other_hook")
    def my_renamed_impl(self, arg1):
        """Implement a hook with a different function name."""

    @hookimpl(optionalhook=True)
    def my_hook(self, arg1, arg2):
        """Don't warn if no matching hookspec exists."""

## PluginManager(project_name)
# The central manager that connects specs with implementations.

pm = pluggy.PluginManager("myproject")

# Register spec classes (defines the contract)
pm.add_hookspecs(MySpec)

# Register plugin instances
pm.register(my_plugin_instance)
pm.register(my_plugin_instance, name="explicit_name")

# Unregister a plugin
pm.unregister(my_plugin_instance)
pm.unregister(name="explicit_name")

# Call a hook (invokes all registered implementations)
results = pm.hook.my_hook(arg1=value1, arg2=value2)
# Returns: list of results from all implementations

# For firstresult=True hooks:
result = pm.hook.my_single_hook(query="something")
# Returns: single value (first non-None result)

# Check if a plugin is registered
pm.is_registered(plugin_instance)

# Get a plugin by name
plugin = pm.get_plugin("plugin_name")

# Get the name of a registered plugin
name = pm.get_name(plugin_instance)

# List all registered plugins
plugins = pm.get_plugins()

# List plugins with their dist-info (packaging metadata)
for plugin, dist in pm.list_plugin_distinfo():
    print(f"{dist.project_name}=={dist.version}")

# Load plugins from setuptools entry points
pm.load_setuptools_entrypoints("myproject")

# Check for and retrieve hook callers
hook_caller = pm.hook.my_hook  # HookCaller instance

# Check if a plugin is blocked
pm.is_blocked("plugin_name")

# Block a plugin from being registered
pm.set_blocked("plugin_name")

# Parse a hook call result into individual results
pm.parse_hookimpl_opts(plugin, "hook_name")

# Get all hook callers
callers = pm.get_hookcallers(plugin)

# Add hook call tracing for debugging
# (undo is a function that removes the tracing)
undo = pm.trace.root.setwriter(print)
pm.enable_tracing()
```

---

## Entry Points Integration

Pluggy can auto-discover plugins registered as setuptools entry points. This is how pytest discovers third-party plugins (e.g., `pytest-cov`, `pytest-xdist`).

### Plugin Side: Declaring Entry Points

In the plugin package's `pyproject.toml`:

```toml
[project.entry-points."myproject"]
my_plugin = "my_plugin_package.plugin:MyPlugin"
```

Or in legacy `setup.py`:

```python
setup(
    name="my-plugin-package",
    entry_points={
        "myproject": [
            "my_plugin = my_plugin_package.plugin:MyPlugin",
        ],
    },
)
```

### Host Side: Loading Entry Points

```python
import pluggy

hookspec = pluggy.HookspecMarker("myproject")
hookimpl = pluggy.HookimplMarker("myproject")

class MyProjectSpec:
    @hookspec
    def process_item(self, item):
        """Process an item."""

def create_plugin_manager():
    pm = pluggy.PluginManager("myproject")
    pm.add_hookspecs(MyProjectSpec)

    # Load all plugins registered under the "myproject" entry point group
    pm.load_setuptools_entrypoints("myproject")

    return pm
```

### Listing Discovered Plugins

```python
pm = create_plugin_manager()

# List plugins that came from entry points (with dist metadata)
for plugin, dist in pm.list_plugin_distinfo():
    print(f"  {dist.project_name} {dist.version}")
    print(f"    plugin object: {plugin}")

# List ALL registered plugins (including manually registered)
for plugin in pm.get_plugins():
    name = pm.get_name(plugin)
    print(f"  {name}: {plugin}")
```

### Blocking Unwanted Plugins

```python
pm = pluggy.PluginManager("myproject")
pm.add_hookspecs(MyProjectSpec)

# Block a plugin BEFORE loading entry points
pm.set_blocked("unwanted_plugin")

# Now load -- unwanted_plugin will be skipped
pm.load_setuptools_entrypoints("myproject")
```

---

## API Quick Reference Table

| Method / Decorator | Description |
|---|---|
| `pluggy.HookspecMarker(name)` | Create a hookspec decorator factory for project `name` |
| `pluggy.HookimplMarker(name)` | Create a hookimpl decorator factory for project `name` |
| `pluggy.PluginManager(name)` | Create a plugin manager for project `name` |
| `pm.add_hookspecs(cls)` | Register a class containing `@hookspec` methods |
| `pm.register(plugin, name=)` | Register a plugin instance |
| `pm.unregister(plugin)` | Unregister a plugin |
| `pm.is_registered(plugin)` | Check if a plugin is registered |
| `pm.get_plugins()` | Get set of all registered plugins |
| `pm.get_plugin(name)` | Get a plugin by name |
| `pm.get_name(plugin)` | Get the name of a plugin |
| `pm.hook.name(kwargs)` | Call a hook (keyword arguments only) |
| `pm.hook.name.call_historic(kwargs=, result_callback=)` | Call a historic hook |
| `pm.load_setuptools_entrypoints(group)` | Auto-load plugins from entry points |
| `pm.list_plugin_distinfo()` | List plugins with packaging dist info |
| `pm.set_blocked(name)` | Block a plugin name from registering |
| `pm.is_blocked(name)` | Check if a plugin name is blocked |
| `pm.check_pending()` | Validate all specs/impls match |
| `pm.enable_tracing()` | Enable hook call tracing for debugging |
| `@hookspec` | Mark a method as a hook specification |
| `@hookspec(firstresult=True)` | Stop at first non-None result |
| `@hookspec(historic=True)` | Replay calls to late-registered plugins |
| `@hookimpl` | Mark a method as a hook implementation |
| `@hookimpl(tryfirst=True)` | Run before other implementations |
| `@hookimpl(trylast=True)` | Run after other implementations |
| `@hookimpl(wrapper=True)` | Yield-based wrapper around other impls |
| `@hookimpl(specname="name")` | Implement a differently-named spec |
| `@hookimpl(optionalhook=True)` | Suppress warning if spec is missing |
